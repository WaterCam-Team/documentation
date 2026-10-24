# LoRaWAN Class B as a future option, and the mDot's power

**Status:** design notes for researchers, 2026-10-23. Nothing here is planned or
built. WaterCam units run the mDot in **Class C** today. This explains what Class C
costs, what firmware changes could lower that while keeping Class C, and what a
**Class B** variant of the whole system would involve, for a deployment that
values battery life over instant remote wake.

Figures marked *estimate* are worked out from datasheets and the firmware, not
measured. Sources are listed at the end.

---

## 1. Why the mDot matters for power

A WaterCam unit is off most of the day. The Witty Pi wakes the Raspberry Pi on a
schedule; between wakes, only the Witty Pi and the mDot draw power. In Class C the
mDot listens continuously, so that a command (above all the remote wake, `!`)
reaches it at any moment.

| | Current | Source |
|---|---|---|
| mDot receiving (Class C), processor active | 43 mA at 3.3 V | MultiTech mDot Developer Guide, electrical characteristics |
| mDot idle, processor active, radio off | 32 mA | same |
| mDot sleep (ST stop mode) / deep sleep (standby) | 40 µA | same |
| mDot transmitting at TXP 2 / 11 / 20 | 34 / 73 / 130 mA | same |
| SX1272 radio receiving, 125 kHz | about 10.5–11 mA | Semtech SX1272 datasheet |
| Witty Pi 4 standby | about 0.5 mA | Witty Pi 4 user manual |
| Voltaic V75 self-consumption in Always On | about 7 mA (26 mW) | Voltaic V75 page |

**Our firmware's draw in Class C, estimated from the firmware: about 15–25 mA.**
- The radio receives continuously: about 11 mA.
- The firmware (Mbed OS 6.8) isn't built tickless. The RTOS idle thread uses plain sleep (WFI) at 96 MHz with deep sleep locked, and the 1 kHz SysTick wakes it every millisecond. The processor then draws a few mA up to about 10 mA, depending on which peripheral clocks run in sleep. ST's table for the related STM32F412 gives 2.9 mA with every peripheral clock off and 17.3 mA with all on, at 100 MHz.
- The rest of the module: about 1 mA.

The mDot runs from the Witty Pi's 3.3 V, which most likely comes from a linear
regulator off 5 V. The battery therefore pays at 5 V:
- **15–25 mA is 1.8–3.0 Wh/day** (5.2 Wh/day at the datasheet's 43 mA);
- a unit's wakes on an hourly daylight schedule use about 1.1–1.4 Wh/day in winter;
- so **the mDot's listening is the largest single item in a unit's energy budget**.

Transmissions don't matter. Even at the edge of coverage, an hourly wake's
LoRa uplinks (DR0, 20 dBm, about 2 s on air) cost under 3 mWh a day.

USB power meters can't measure this: they show nothing below 10–20 mA. Use a
multimeter in series with the mDot's VDD, with the Pi off. The detailed budget
is in SU-WaterCam `docs/POWER_ANALYSIS.md`.

---

## 2. Lowering the draw while keeping Class C

The Dot library won't sleep the processor while the radio receives (`mDot::sleep()`
returns `MDOT_NOT_IDLE`), so Class C can't use the mDot's 40 µA stop mode. The radio's
11 mA is the floor. What can be cut is the processor's share. Each change below
keeps every function: instant remote wake, the Pi watchdog, the emergency press
and the AT interface. Each should be built, flashed and measured on its own.

| Change | Expected saving | Risk |
|---|---|---|
| **Sleep clocks.** STM32F4 peripherals keep their clocks in sleep unless their bit in `RCC_AHB1LPENR` / `APB1LPENR` / `APB2LPENR` is cleared. Clear the bits of everything not needed while waiting: unused GPIO ports, DMA, CRC, ADC, I2C, unused timers, the flash interface. Keep the AT UART (USART2), the microsecond-ticker timer and the GPIO ports with interrupts (the radio's DIO lines, PA_6). Clocks come back on by themselves when the processor wakes. | about 1–5 mA | Low. A peripheral wrongly cut stops working only while asleep; test the AT port, the radio, the emergency input and the watchdog. |
| **Lower system clock**, 96 → 32 or 48 MHz, with the regulator's lower voltage scale. Sleep current scales roughly with the clock. Set in Mbed's `system_clock.c` for the target. | about 2–5 mA | Medium. The Dot library is a binary; its timing uses Mbed timers, so it should adapt, but join, receive windows, Class C reception, the UART baud rate and SPI to the radio all need regression tests. MultiTech documents no lower-clock setting. |
| **Tickless idle** (`MBED_TICKLESS`): no SysTick wake every millisecond. | about 0.5–1 mA | Medium. It changes the RTOS's timing base. |
| **Event-driven AT loop.** The terminal now polls every 10 ms. | small; only worth it with tickless | Low. |

**Together:** perhaps 13–15 mA instead of 15–25 mA, saving about 0.2–1.2
Wh/day per unit. Measure the baseline first: if the real figure is near 15 mA,
the gain is small.

Hardware options don't affect functionality either:
- **A switching regulator** (to about 3.6 V) instead of a linear one for the mDot saves about a quarter of its share.
- **Feeding it 5 V directly** (pcb-designs `Schematic_Remaining_Work.md` #20) improves the supply margin but not the energy: the mDot's own regulator is linear.

**Beyond this, Class C can't go.** Lower power needs the radio to listen less, which
means Class B or Class A.

---

## 3. A Class B variant of the whole system

### 3.1 What Class B changes

In Class B the gateways transmit a **beacon** every 128 s, timed by GPS. Devices
lock onto it and open short receive windows, **ping slots**, at agreed times. Between
them the radio is off and the processor can sleep.
- A downlink (a command, or the remote wake `!`) waits for the next ping slot rather than arriving at once.
- Periodicity n = 0–7 gives 2^(7−n) slots per beacon period: from one a second to one every 128 s.

| Ping period (`AT+PP`) | Slots per 128 s | Worst-case downlink delay | Radio listening |
|---|---|---|---|
| 7 | 1 | 128 s | beacon + 1 slot per 128 s |
| 5 | 4 | 32 s | beacon + 4 slots |
| 3 | 16 | 8 s | beacon + 16 slots |
| 0 | 128 | 1 s | beacon + 128 slots |

*Estimate.*
- **Radio:** the beacon (once per 128 s) and a few ping slots of tens of milliseconds each average under 0.1 mA.
- **Processor:** dominates.
  - If it can stop between slots (ST stop mode, 40 µA), the mDot could average **roughly 0.2–1 mA, about 0.02–0.1 Wh/day**: over 90% less than Class C.
  - If it stays in plain sleep as now, it's still 4–11 mA.
  - The Dot library documents automatic stop-mode sleep between receive windows for Class A; whether it does so in Class B isn't documented and has to be tested.

### 3.2 What the system needs, layer by layer

**Gateways**
- Class B needs gateways that **transmit beacons**: a GNSS (GPS) time source with a pulse-per-second line, and a packet forwarder configured to send them.
  - In US915, beacons go out at DR8 (SF12, 500 kHz), hopping over 8 channels from 923.3 MHz in 600 kHz steps.
- ChirpStack doesn't generate beacons; the gateway does.
- For MultiTech Conduit-class gateways we found no MultiTech document saying mPower or its packet forwarder emits Class B beacons. A ChirpStack forum thread says a Conduit without GPS can't.
- **Check this first:** the gateway's model and whether it has a GPS card, and ask MultiTech whether its forwarder sends beacons. ChirpStack Concentratord documents a `[gateway.beacon]` section for SX1301 concentrators with GNSS.
- Every gateway a unit might hear needs to beacon, or the unit's lock depends on one gateway.

**Network server (ChirpStack v4)**
- Device profile: enable Class B, and match the ping-slot settings to the devices.
- `[regions.network.class_b]` holds `ping_slot_dr` and `ping_slot_frequency`; the US915 default is DR8 with hopping. `[network.scheduler] class_b_schedule_advance` defaults to 5 s.
- The device needs the time to find the beacon (DeviceTimeReq, `AT+GPSTIME`); check that ChirpStack answers it.
- Downlinks queued for a Class B device go out in its next ping slot automatically.
- ChirpStack changed Class B scheduling in v4.19; test on the version in use.

**mDot firmware**
- The Dot library supports Class B (LoRaWAN 1.0.3/1.0.4, library 3.2.1 and later):
  - AT commands: `AT+DC=B`, `AT+PP=0..7`, `AT+BLS` (beacon lock), `AT+BTO` (Class B ack timeout), `AT+GPSTIME`;
  - library calls: `setClass("B")`, `setPingPeriodicity()`, `getGPSTime()`;
  - events: `BeaconRx` and `BeaconLost`.
- Our firmware has these commands, but its packet processor forces Class C at every start. A Class B build would make the class a setting.
- After beacon lock the device must send an uplink, so the network knows it's in Class B.
- The library keeps Class B for up to 2 h (7200 s) without beacons, then falls back to Class A.
- A MultiTech forum report (library 3.2.1) describes a beacon that never locks; test on the 4.1.x library in use.
- Keep the AT port usable while the Pi is on. The processor can't receive UART characters in stop mode, so sleep only while the Pi is off: the firmware already reads the Pi's power state on PA_6.
- Everything else stays as it is: the remote wake (`!` → PB_1 press), the Pi watchdog, the self watchdog and serial-silence restart, the emergency input. They just act after the next ping slot.

**Node software (SU-WaterCam)**
- Little changes. The Pi still sends its data through the mDot's AT port while awake.
- When the Pi is on, its uplinks also give the server a chance to reply in the receive windows after them, as in Class A.
- Schedules, emergency mode and IP transport are unaffected.

**API and dashboard**
- Command delivery (`app/node_delivery.py`) currently waits 60 s for an answer to the remote wake before resending. With Class B it should allow for the ping period plus the Pi's boot.
- The command panel should say "arrives within about N s" for Class B units.
- A per-unit setting (device class and ping period) tells the API what latency to expect.

### 3.3 A heartbeat with a server reply

Class B depends on infrastructure (beacons) that can fail silently, and a unit whose
Pi is off says nothing for hours. A **heartbeat sent by the mDot itself** fixes both.
It's worth having in any class, but Class B needs it most.

- **Uplink.** Every N minutes (a setting, e.g. `AT+HB=<minutes>`, 0 = off; 60 is a reasonable default), unconfirmed, on its own LoRaWAN port so the API can tell it from the Pi's data. It fits DR0's 11 bytes:

  | Bytes | Field |
  |---|---|
  | 1 | type / version |
  | 1 | flags: Pi powered (PA_6), beacon locked, current class, last command applied |
  | 1 | heartbeat sequence |
  | 2 | uptime, hours |
  | 1 | last reset reason (power, watchdog, serial silence, crash) |
  | 1 + 1 | last beacon RSSI and SNR |
  | 2 | battery: the V75's charge signal on the mDot's ADC, once wired (the SBU pins carry half the cell voltage, 1.6–2.1 V) |
  | 1 | spare |

- **Server reply.** The API answers each heartbeat with a short downlink:
  - the sequence echoed;
  - flags: commands pending, wake the Pi now;
  - optionally the server time.
  - In Class B it arrives in the next ping slot. In Class A or C it can go in the receive windows after the heartbeat only if it's queued fast enough: ChirpStack waits briefly for the application before scheduling them, and a longer RX1 delay gives more time. Test the timing. Otherwise it waits for the next heartbeat.
  - A confirmed heartbeat every few times gives a network-level acknowledgement independent of the API.
- **What the mDot does without replies:**
  - after 3 missed: rejoin (the session may be lost);
  - after more, and at most once a day: press the Pi's switch so the Pi can report over WiFi or cellular.
  - Count the misses in a status command.
- **What the server does with heartbeats:**
  - store the last one per unit;
  - show "radio alive" and the beacon lock on the dashboard;
  - alert after 2 missed heartbeats. That catches a unit that's gone quiet while its Pi is off, which nothing catches today.
- **Cost:** about 0.04 mWh per heartbeat at DR0 and 20 dBm, about 1 mWh a day hourly.

### 3.4 Comparison

| Mode | mDot draw (estimate) | Battery, per day | Remote wake / command delay | Needs |
|---|---|---|---|---|
| Class C, today | 15–25 mA | 1.8–3.0 Wh | about 1–2 s | nothing new |
| Class C, firmware tuned (section 2) | 13–15 mA | 1.6–1.8 Wh | about 1–2 s | firmware changes, tests |
| **Class B**, ping every 32 s, stop between slots | 0.2–1 mA | 0.02–0.1 Wh | ≤ 32 s | beaconing GPS gateways, firmware, server settings, test of stop mode |
| Class B, plain sleep between slots | 4–11 mA | 0.5–1.3 Wh | ≤ 32 s | the same, minus stop mode |
| Class A + hourly heartbeat, stop between | about 0.05–0.2 mA | about 0.01–0.03 Wh | up to an hour (the heartbeat interval) | firmware only |

**Class A with a heartbeat** is the simplest low-power option: no gateway changes,
the mDot asleep except for its heartbeat and the receive windows after it. It suits
deployments that don't need a fast remote wake.

**Class B** sits between: tens of seconds of latency for a few percent of Class
C's energy, at the price of GPS-disciplined, beaconing gateways.

For a unit on a 72 Wh pack and a 20 W panel, the saving matters most in winter.
There, a unit's total use of about 3.5–5 Wh/day could drop to about 1.5–2 Wh/day,
roughly doubling how long it lasts without charge.

### 3.5 Suggested order, if a project takes this up

1. Measure the mDot's current in Class C (multimeter in series with VDD, Pi off) to fix the baseline.
2. Confirm gateway beacon support (GPS card, packet forwarder), since this is the deciding constraint for Class B.
3. Add the heartbeat and server reply in Class C first. They're useful on their own, and they're the safety net Class B needs.
4. On one bench unit, try Class B:
   - beacon lock time;
   - ping-slot downlink delay;
   - remote wake through a ping slot;
   - whether the processor stops between slots (the measured current tells you);
   - recovery when beacons stop.
5. Make the device class a per-unit setting in firmware, API and dashboard, defaulting to Class C.

### 3.6 Open questions

- Do our gateways have GPS, and does their packet forwarder send Class B beacons?
- Does the Dot library 4.1.x stop the processor between Class B ping slots?
- Does ChirpStack (the version in use) answer DeviceTimeReq, and how reliably does it schedule ping-slot downlinks?
- What remote-wake delay can a deployment accept: seconds, a minute, an hour?

---

## Sources

- MultiTech mDot Developer Guide (S000612), electrical characteristics: https://static.chipdip.ru/lib/337/DOC028337942.pdf (distributor copy)
- MultiTech mDot AT Command Guide (S000643), Class B and sleep commands: https://www.tessol.com/data/item/1687250545/1687251217_6736_05_S000643mDotATCommandGuide.pdf (distributor copy)
- MultiTech Dot library changelog: https://github.com/MultiTechSystems/libmDot/blob/master/CHANGELOG.md
- MultiTech xDot/mDot firmware PCN (4.1.38): https://multitech.com/wp-content/uploads/PCN-03092026-001_xDot-mDot_FW_Updates.pdf
- ChirpStack device classes: https://www.chirpstack.io/docs/chirpstack/features/device-classes.html
- ChirpStack configuration: https://www.chirpstack.io/docs/chirpstack/configuration.html
- ChirpStack Concentratord configuration: https://www.chirpstack.io/docs/chirpstack-concentratord/configuration.html
- ChirpStack forum, Conduit and Class B: https://forum.chirpstack.io/t/using-multitech-conduit-for-class-b/2894
- Voltaic V75: https://voltaicsystems.com/v75/ and Always On: https://voltaicsystems.com/always-on/
- STM32F412 datasheet, sleep-mode current (used as a stand-in for the F411): https://www.alldatasheet.com/html-pdf/2178744/STMICROELECTRONICS/STM32F412RGU6PTR/219161/96/STM32F412RGU6PTR.html
