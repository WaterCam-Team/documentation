# Plan: command routing, phase B

**Written:** 2026-10-20. To start after the phase A observations below.
The design is `COMMAND_ROUTING_DESIGN.md`; this is the checklist for doing it.

## Before starting: a week of real observations

Phase A (API #169, observe only) has been in production since 2026-10-17. Since
then 005 has been always on at the bench, 006 held, and 008–011 switched off, so
nothing yet shows how units on a field schedule look. Phase A needs **at least a
week with units on their deployment schedules** (006 and some of 008–011 once
deployed), then:

- [ ] For each scheduled unit, compare the panel's LoRa-active and IP-active with
      what the unit did (Witty Pi logs, wake times). Any wrong state is a window
      or counting bug.
- [ ] Compare the computed windows with the real schedules: is 1.5 × the median
      daily longest gap right for a 09:00–17:00 two-hourly schedule with a
      16-hour overnight gap? Adjust the constants in design 4.3 if not.
- [ ] The DR distribution from `lora_dr` in Influx: how many uplinks are DR0
      (11-byte downlinks). That sets which commands need IP in practice
      (design 11).
- [ ] How often a remote start (`!`) or a configuration change was needed, and
      whether it reached the unit; use those as the ledger's first test cases.
- [ ] Answer anything still open in design section 19 that the data settles.

## Phase B work (API), each a PR with tests (design 17.1)

1. **Command ledger** (design 13.1): SQLite table and Alembic migration;
   `submit`, `on_ack`, expiry and superseding (9.1, 9.2); `node_delivery`'s
   progress mirrored from it. No routing change yet.
2. **`transport_type` becomes an override** (13.2): migration to `auto`, keeping
   explicit overrides; the dashboard shows "auto (observed: …)".
3. **The router, `app/routing.py`** (13.3), behind `COMMAND_ROUTING=legacy|observed`
   (default `legacy`): the decision table in section 6, held commands (7),
   fallback timing (8). With `legacy` it only logs what it *would* do, beside
   what was done.
4. **`node_delivery` and the IP queue on the router** (13.4, 13.5); doubtful
   counting and the probe (4.4).
5. **Dashboard** (14): the Transport section, per-command results, held and
   queued commands; axe checks as in 14.4 and the cross-browser pass.
6. **Alerts** (13.7) for doubtful units and expired commands.

## Turning it on

- [ ] Deploy with `COMMAND_ROUTING=legacy`; check the migration result
      (`transport_type` all `auto` apart from deliberate overrides).
- [ ] A few days of "would route" logs compared with what was done; fix
      surprises.
- [ ] Switch to `observed`; watch the ledger for a few days. `legacy` is the
      rollback.

## After phase B

Phase C (sequenced frames and acks, design 10 and 12) needs a node release and
the bench tests B1–B10 on 005 (design 17.4) before any field unit gets it.
