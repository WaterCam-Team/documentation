# Segmentation model updates over the air: system plan

Status: plan, 2026-10-09. Nothing here is built yet. The measurements it relies on are in
`MODEL_OTA_DIFF_FEASIBILITY.md` (diff sizes) and `SAM_ON_PI_FEASIBILITY.md` (Pi timings).

## Goal

After a node is deployed, a user can pick a validated model release on the dashboard and
push it to one node or a selection of nodes. The node receives the smallest practical
file. It rebuilds a model that is **bit-identical** to the release built on the
workstation, proves that it did, and can go back to the previous model at any time.

**Assumption.** Before deployment, every node gets a full release over USB (release
`R1`), built by the annotation system. Over-the-air updates only ever start from a
release the server knows exactly.

**What "matches" means.** The release is defined as the *reconstructed* model: the base
plus the quantized delta, exactly as the node will compute it. It is not defined as the
raw output of training.
- The workstation validates that reconstructed model.
- The node must produce a file with the same SHA-256.
- How faithful the reconstruction is to the newly trained model is a validation gate
  (fidelity, below), not something left to chance.

## Design decisions

1. **A release tree (one parent per release), transmitted as a chain of deltas.**
   - There is a global trunk, plus optional regional branches (see "Regional fine-tunes").
     Every release has exactly one parent.
   - Each release `Rn+1` is defined as `Rn + Q(Tn+1 - Rn)`, where `Rn` is its parent. `Tn+1` is the newly trained
     model, and `Q` is a per-tensor quantizer (int8 or int4, with zero tensors skipped).
   - Every node on `Rn` therefore gets the same `Rn+1`, so there is one identity to
     validate and verify per release.
   - A node several releases behind applies the chain `Rk -> Rk+1 -> ... -> Rn` and checks
     each step's hash. If the chain would be larger than the full compressed file, it gets
     the full file instead.
   - The server never computes deltas: it stores what the workstation built and only
     serves files. It needs no ML dependencies.
2. **Train the next model from the released weights, not from the last trained ones.**
   Fine-tuning starts from `Rn`'s reconstructed weights, with `--stats` pinned to `R1`'s
   normalization. Otherwise the delta grows and the graph changes.
3. **Make updates small by how they are trained.** Measured payloads for the fp32 model
   (15.1 MB raw, 13.7 MB compressed):

   | Training mode | Delta | Fidelity to the trained model |
   |---|---|---|
   | Head-only | int8: **288 KB**; int4: **82 KB** | water IoU 0.9998 / 0.993 |
   | Full fine-tune | int8: 3.2 MB; int4: **1.2 MB** | 0.9999 / 0.998 |
   | LoRA rank 8 + head | about 0.15 MB + head delta (sizes only; accuracy not measured) | — |
   | Full replacement | 13.7 MB | for architecture or normalization changes |

   The release builder tries encodings from smallest to largest and ships the smallest one
   that passes the fidelity gate.
4. **Zero new node dependencies for the payload path.** The package uses `lzma` from the
   standard library (xz compressed within 1% of zstd in the measurements) plus numpy and
   onnx, which are already in the node's `5band` env. Signature verification needs one
   addition, installed at USB provisioning (see Security).
5. **Keep old releases on the node.** Rollback becomes a pointer switch with zero bytes
   transferred. Reverse deltas would not reproduce old releases exactly, and full
   re-downloads cost about 14 MB, so neither is used for rollback.
6. **Pull, don't push.** Nodes are off most of the time and only reachable on their own
   schedule. The dashboard creates a *deployment*; each node discovers it at its next IP
   wake, downloads it (resuming across wakes if needed), and reports each step.

## Release package format (`.wcmu`)

A single file: a signed JSON manifest followed by an lzma-compressed payload.

```
manifest (canonical JSON, Ed25519-signed):
  format: 1
  release_id: "seg-g0007" | "seg-<region>-0003"   # monotonic per track
  kind: "delta" | "full"
  lineage: {track: "global" | "region", region_id, global_base: {release_id, sha256}}
           # global_base = the trunk release this one descends from (itself, for a trunk release)
  parent: {release_id, sha256}        # model file the delta applies to (delta only)
  result: {sha256, size}              # model.onnx after applying: THE identity check
  payload: {sha256, size, compression: "xz"}
  encoding: {scheme: "tensor-delta-v1", bits: 4|8, per: "tensor"}
  contract: {arch, modality, bands, classes, input: {...}, normalization_sha256}
  runtime: {onnxruntime: "1.19.2", numpy: "1.24.1"}     # what the fingerprint assumes
  fingerprint: {seed, input_shape, mask_sha256, logits_sum}  # recorded on the bench Pi
  validation: {report_sha256, gold_water_iou, fidelity_water_iou, bench_ms, ...}
  created_at, built_by, signer_key_id
signature: base64 Ed25519 over the canonical manifest bytes
```

**Payload (`tensor-delta-v1`).** For each changed ONNX initializer, by name:
`dtype`, `shape`, `scale` (float32), and `q`, packed as int8 or two int4s per byte.
Unchanged tensors are omitted.

**Reconstruction rule (defined once and used everywhere).** Per element,
`w = float32(float32(q) * scale)`, then `w_new = float32(w_old + w)`. These are separate
numpy elementwise operations: IEEE-754 single precision, round-to-nearest, no fused
multiply-add. The node rewrites the initializers and serializes the ONNX file in the same
way the workstation does.

**Phase 0 must prove that the resulting bytes are identical on x86-64 and aarch64.** If
they are not, reconstruction moves to an integer-only rule that is still exact: transmit
`q` and an exponent-aligned integer scale and add in the integer domain.

## Components to build

### 1. Shared library `model_package.py` (canonical copy in `SU-WaterCam/tools/`)

The node is the tightest constraint (Python 3.9, numpy 1.24, no extra wheels), so the
canonical code lives with the node. `photo_processing` imports it by path. A test in each
repo fails if the copies differ.

API:
- `build_delta(parent_onnx, trained_onnx, bits) -> (package_bytes, result_onnx_bytes)`
- `build_full(onnx)`
- `read_manifest(pkg)`
- `verify_signature(manifest, sig, pubkeys)`
- `apply(pkg, parent_path, out_path) -> result_sha256`, which checks the parent hash
  before and the result hash after
- `fingerprint(onnx_path, seed) -> {mask_sha256, logits_sum}`

Unit tests:
- round trip;
- a tampered payload, wrong parent, or bad signature is rejected;
- a fixture (base plus package) whose expected `result.sha256` is recorded on both x86 and
  the Pi.

### 2. Release builder in the annotation system (`photo_processing`)

**Release registry.** `annotator/work/releases/`, with one directory per release:
- `model.onnx`, the HF weights that reproduce it, `deploy.json`;
- `package_from_<parent>.wcmu`;
- `validation_report.json`;
- `lineage.json`: parent, training config, gold-set manifest hash, stats hash, and a git
  commit for each repo.

**Training modes.** `--from-release Rn` with modes `full`, `head-only` and `lora:r`.
- `--stats` is pinned to `R1`'s automatically.
- The head-only freeze is the `segformer.*` prefix freeze used in the 2026-10-09
  experiment.
- LoRA needs a deterministic merge: accumulate the rank-r outer products elementwise in
  float64 in a fixed order, then round once to float32. Never use BLAS matmul here, since
  its summation order differs across platforms.

**Weight spaces.** Revised after Phase 0.
- **The ONNX file is canonical.** Deltas are always computed and applied in ONNX
  initializer space, and the release identity is the ONNX file's sha plus its weights
  digest.
- Phase 0 found that 202 of 208 HF parameters map bit-exactly onto ONNX initializers (150
  as-is, 52 transposed).
- The exporter folds the decode head's `linear_fuse` conv and `batch_norm` (weight, bias,
  running mean and variance) into two computed initializers (`onnx::Conv_1648`/`1649`).
  "Apply in HF space, then export" therefore differs from the ONNX-space reconstruction in
  exactly those two tensors.
- **Getting HF weights back for the next training run.**
  - Copy the 202 mapped tensors back exactly.
  - For the fused pair, set `linear_fuse.weight` to the fused weight, and BatchNorm to
    weight 1, bias equal to the fused bias, mean 0, variance 1. That is equivalent up to a
    factor of 1/sqrt(1 + eps), which is harmless because training moves these weights
    anyway.
  - The next delta is then taken between the exported trained model and the release ONNX,
    so the approximation never reaches a node.
- **Alternative,** if exact lockstep with HF is wanted for auditing: export with
  BatchNorm left unfolded, so all parameters map one-to-one. That is a graph change, so it
  means one full release; check the speed cost on the Pi first.

**Build release.** A Train-panel action. Steps:
1. Export the candidate.
2. Try encodings: int4 head-only, int8, int4 full, int8 full, then a full file.
3. Reconstruct each and run gates 0-2 (below), picking the smallest passing encoding.
4. Run gate 3 on the bench Pi.
5. Sign.
6. Write the registry entry.
7. Upload the release to the API.

**USB provisioning bundle.** `--provision`. Contents:
- the full release `R1` (or the current one) laid out in the node's directory structure;
- the signing public key(s);
- the node updater code;
- a `provision.sh` that installs it.

This replaces the legacy mmseg `iter_100` model on the node, which can't be diffed
against.

### 3. Node (`SU-WaterCam`)

**Model store.**

```
/home/pi/models/segformer/
  releases/<release_id>/model.onnx, manifest.json, manifest.sig, deploy.json
  current  -> releases/<id>     # symlink, swapped with rename(2) after fsync
  previous -> releases/<id>
  pinned   -> releases/R1       # factory model, never pruned
  base     -> releases/<id>     # current's global_base (the trunk release it branched from);
                                # equal to current on the trunk. Kept so switching region or
                                # falling back to global needs at most one small forward delta
  staging/                      # partial downloads (resumable), apply scratch
  state.json                    # updater state machine, survives power loss
```

Keep `current`, `previous`, `pinned`, `base`, and up to 2 more releases. Pruning never
removes an ancestor of `current` that is the newest trunk release on the node. At about 15 MB each this
is trivial against roughly 40 GB free.

**`segformer_daemon.py` changes.**
- Default `--model` becomes `/home/pi/models/segformer/current/model.onnx`. The service
  file currently points at `segformer_5band_int8.onnx`, which doesn't exist on 005, so it
  silently falls back to fp32.
- New socket commands:
  - `{"cmd": "info"}` returns release_id, sha256 and load time;
  - `{"cmd": "reload"}` reloads `current` in place, so pi needs no privilege to restart a
    service;
  - `{"cmd": "selftest", "seed": ...}` returns the fingerprint.
- Report `release_id` and `inference_ms` with each result.

**Trial mode.**
- A newly activated release is in `trial` state in `state.json` until the first real
  inference succeeds within the latency budget.
- If the daemon fails to load it, or `N` (= 2) inferences error out, the daemon or
  updater reverts `current` to `previous` and records why.
- A systemd `ExecStartPre` guard does the same if the daemon crash-loops at boot.

**`tools/model_update.py`, launched by a new SQify step in `ticktalk_main.py`.**
- The step only launches it: it runs `~/miniforge3/envs/5band/bin/python
  tools/model_update.py` under `systemd-run` with the memory cap. It never imports onnx
  itself.
- All model-file work (apply, hash, self-test with ORT) happens in the 5band env. That env
  has onnx 1.19.1, ml_dtypes, protobuf 6.33.6 and ORT pinned in
  `segformer_5band/5band_environment.yml`, and it is where Phase 0 proved bit-identical
  reconstruction on the Pis (numpy 1.24.1).
- The ticktalk venv (`SU-WaterCam/requirements.txt`, numpy 2.2.4) therefore doesn't need
  onnx. Decided 2026-10-10. It runs after
segmentation and the IP upload, so an update never delays data. It is followed by a
pickle rebuild (compile and commit both pickle files). It runs only when:
- `model_update.enabled` is set in `runtime_config.json`;
- the IP link is up;
- the remaining awake-window and battery thresholds allow it (`model_update.min_vout`,
  `model_update.max_seconds`).

**Updater state machine** (one step per wake if time is short; every step resumable and
idempotent):

```
idle -> manifest_ok      GET /ip/model/{device}/pending; verify signature, first parent sha is a release
                         on disk (current, base, previous or pinned),
                         contract matches, release not revoked, free disk >= 3x size
     -> downloading      GET package with HTTP Range into staging/; resume next wake if cut
     -> payload_ok       sha256(payload) == manifest.payload.sha256
     -> applied          apply chain step(s) into staging/; sha256(result) == manifest.result.sha256
     -> selftest_ok      load in a separate process (memory-capped, see below), fingerprint == manifest
     -> activated        move into releases/, previous := current, current := new (atomic), daemon reload
     -> trial            shadow check: run previous and new on the newest real capture,
                         report agreement and water fraction (one extra 1.6 s inference)
     -> confirmed        after the first K (= 3) successful real inferences
any failure          -> report reason; discard staging; current unchanged (or reverted if after activation)
```

Each transition is POSTed to `/ip/model/{device}/status`. If the post fails, it is queued
and sent next wake.

**Lesson from 2026-10-08.** Anything heavy on the node runs under
`systemd-run -p MemoryMax=2800M -p MemorySwapMax=0`. Static calibration once made 005 run
out of memory until the watchdog reset it. Applying a package is light (one 15 MB model in
numpy), but the self-test loads an ORT session alongside the running daemon.

**Version reporting.**
- Every IP uplink adds `model_release` and the first 8 hex characters of `model_sha256`.
- Optionally, LoRa status carries a 1-byte release number. That needs an API decoder
  change, so it is Phase 5.

**Signature verification.** Revised after Phase 0.
- Use a **vendored pure-Python Ed25519 verify** (RFC 8032 reference code, about 100
  lines) as the primary path. On 005 it verifies in 16 ms, passes the RFC test vector and
  rejects a tampered manifest, with zero new dependencies.
- `cryptography` is optional. Its wheels (50.0.2, with cffi, pycparser and
  typing_extensions) install in 10 s and take 18 MB, and verify in 1.2 ms. That speed
  doesn't matter for one verify per update. The public key is
in `config/model_signing_pubkeys.json` and installed at provisioning.

### 4. API server (`API`)

**Tables** (Alembic migration):
- `model_releases`: id, release_id, parent_id, kind, sha256, size, contract json,
  validation json, signer, created_at, status (`draft` / `approved` / `revoked`),
  revoked_reason.
- `model_packages`: release_id, from_release_id (null for full), payload path, sha256,
  size.
- `node_model_state`: device_id, current_release, current_sha, previous_release,
  `releases_on_disk` (list of sha from the node's reports), last_report_at,
  trial/confirmed.
- `model_regions`: id, name, description, created_at.
- `devices.model_region_id`: nullable; null means the node follows the global trunk.
  Optional `neighborhoods.model_region_id` gives a default for devices in that
  neighborhood. Regions are kept separate from neighborhoods: neighborhoods group gauge
  triggers and are finer than a region of similar scenes would be.
- `model_releases` also gets `track` and `region_id` (null for the trunk) and
  `global_base_id`.
- `model_deployments`: id, release_id, created_by, created_at, note, canary_device_ids.
- `model_deployment_targets`: deployment_id, device_id, state, bytes_sent, last_step,
  last_error, history json.

**Endpoints.** Users use session auth plus CSRF, with new permissions `manage_models` and
`deploy_models`.
- `POST /models/releases`: workstation uploads the manifest, signature and packages. The
  server verifies the signature and stores the files (`app/storage.py`).
- `GET /models/releases`, `GET /models/releases/{id}`: release list and validation report.
- `POST /models/releases/{id}/approve`, `.../revoke`.
- `POST /models/deployments`: target device_ids and a canary subset.
- `POST /models/deployments/{id}/cancel`.
- `POST /devices/{id}/model/rollback`, with a target of `previous`, `pinned` or a
  release_id already on the node.

**Node endpoints.** They use the same reach as `/ip/uplink`, which has no device auth today
and relies on Tailscale ACLs.
- `GET /ip/model/{device_id}/pending` returns the manifest(s) for the chain from that
  node's reported release, or a rollback command.
- `GET /ip/model/packages/{pkg_id}` with HTTP Range support.
- `POST /ip/model/{device_id}/status`.

Integrity doesn't depend on transport auth, because nodes only accept signed packages.
Status reports could be spoofed from inside the tailnet. If that matters, add a per-node
HMAC key at provisioning (Phase 5).

**Chain selection.** Packages exist only along forward edges of the release tree, and the
node keeps several releases on disk.
- The server takes every release on the node (`releases_on_disk`) that is an ancestor of
  the target, and serves the smallest forward chain from any of them.
- Example: a node on `seg-north-0003` (branched from `seg-g0007`) moving to
  `seg-south-0002` (also branched from `seg-g0007`) gets the single small package
  `seg-g0007 -> seg-south-0002`, applied from its `base`.
- It serves the full package instead if there is no forward path from anything on the
  node, if the chain is larger, or if the node's sha is unknown. It refuses if the node isn't on a known release (for example, still on legacy
`iter_100`): that node needs USB provisioning.

**Dashboard.**
- **Releases page:** lineage, validation report, size per parent, approve and revoke.
- **Deploy dialog:** pick nodes (or a whole region), see each node's region, current
  release and the bytes it would download, and set a canary. Deploying a regional release
  to a node outside that region needs an explicit override. "Update to latest for its
  region" is the default action.
- **Regions page:** create regions, assign nodes (or set a neighborhood default), and see
  which release each region is on.
- **Progress per node:** state, bytes, the last step and when, and badges:
  - "verified weights" (result sha matches);
  - "verified outputs" (fingerprint matches);
  - "confirmed" (trial passed).
- **Rollback button** per node and per release.
- **Alert** when post-deploy monitoring flags a node.

**Data budget guard.** Per-node bytes used by model updates are recorded. The deploy
dialog warns above a configurable size (default 2 MB) and needs an override above 5 MB.
Context: the SIM cap is 500 MB lifetime, a full model is 2.7% of it, and Tailscale already
costs 126-474 KB per wake.

## Validation system (before a release can be approved)

Every gate runs on the **reconstructed** model, whose sha is the one in the manifest.

**Gate 0, contract.**
- The graph matches the parent with initializers removed, unless `kind: full`.
- The input is 5 bands, NCHW, raw 0-255.
- Classes and normalization sha are unchanged.
- `accepts_node_shape` holds (from `deploy.py`).
- `deploy.json` is complete.

**Gate 1, quality against the gold masks.**
- Leave-one-session-out held-out evaluation (`training/cv.py`), never a random split.
- Reports water IoU, precision, recall and boundary F1, per site and overall.
- Pass if:
  - overall water IoU >= the current approved release's, minus 0.01;
  - no site with >= 5 gold scenes regresses by more than 0.02;
  - the false-water rate on no-water scenes doesn't rise.
- With fewer than 5 gold scenes at a site, mark it "insufficient evidence" and require a
  manual override. Measuring dataset size and site coverage is part of the report.

**Gate 1R, regional releases only.**
- Same evaluation, restricted to the region's held-out gold scenes, still leave one
  session out.
- The regional release must beat the global release it branched from on that region by
  at least 0.02 water IoU. If it doesn't, the region simply stays on the trunk.
- It must not regress on the region's no-water scenes.
- It must not collapse elsewhere: water IoU on other regions' gold scenes stays within
  0.10 of its global base. That is a sanity check against overfitting, since nodes do get
  moved.
- Minimum evidence: at least 20 gold scenes from at least 3 capture sessions in the
  region. Below that, a regional fine-tune is refused, because a dozen scenes from one
  week is how overfitting gets shipped.

**Gate 2, fidelity.**
- Reconstructed against trained model on held-out captures.
- Pass if water IoU >= 0.995 and pixel agreement >= 0.999.
- Used to pick the smallest encoding that passes.

**Gate 3, bench Pi (005).** Under a memory cap, using the node's own `model_update.py`:
- apply the actual package to the actual parent;
- the result sha must equal the manifest;
- p50 inference <= 1.25x the current release and <= 3 s, peak RSS < 1 GB, `get_throttled`
  clean;
- record the ARM output fingerprint into the manifest before signing. Deployed nodes with
  the same ORT and numpy versions should then reproduce it exactly.

**Gate 4, human approval** in the dashboard, with the report attached.

**Rollout gate.**
- Canary node(s) first.
- The rest of the selection is held until canaries reach `confirmed`.
- Any canary failure or rollback pauses the deployment.

## Rollback

**Automatic, on the node:**
- before activation, any failed check means staging is discarded and nothing changes;
- after activation, a load failure, inference errors, a blown latency budget or a
  crash-looping daemon revert to `previous`;
- either way the node reports the reason.

**Manual, from the dashboard:**
- `rollback` to `previous`, `pinned` (`R1`), `base` ("revert to global") or any release
  still on the node is a pointer switch plus `reload`, with zero bytes downloaded;
- the node confirms by reporting the new `current_sha`.

**Fleet:**
- revoking a release blocks new deployments of it and cancels pending ones;
- nodes check revocation when they poll for pending updates;
- the dashboard offers "roll back all nodes on this release".

**Post-deploy monitoring** (server side, from data already uplinked):
- per node, compare water fraction and the rates of all-water and no-water frames over
  the first days after activation with the same node's history on the previous release;
- flag shifts beyond a threshold, which are alerts, not automatic rollback;
- trial-period shadow agreement below 0.9 is flagged immediately.

## Regional fine-tunes

Different deployment regions (terrain, water colour, vegetation, snow, camera mounting)
may each be better served by their own fine-tune. The plan supports this without a
separate system.

**Shape of the tree.**

```
R1 (USB) -> seg-g0002 -> seg-g0003 -> ... (global trunk, trained on all gold data)
                 \               \
                  seg-north-0001  seg-north-0002 (rebased onto g0003)
                  seg-south-0001
```

**Branches should be head-only or LoRA by default.**
- A region shares the trunk's encoder and adds its own decode head or low-rank adapter.
- Measured head-only deltas are 82-288 KB.
- Switching a node between regions, or back to global, is one such package applied from
  the shared `base`, or a zero-byte pointer switch if the target is already on disk.

**Trunk updates fan out.** When `seg-g0008` is approved, every active region gets a
rebase:
- the region's model is fine-tuned again from `seg-g0008` with the region's data,
  producing `seg-<region>-000k`, whose parent is `seg-g0008`;
- it goes through all gates, including 1R against `seg-g0008`.

Nodes on the old branch reach it by forward edges: `base (g0007) -> g0008 -> region-000k`.
That is two packages, or one if the node is moved to g0008 first. The workstation can also
build a direct `g0007 -> region-000k` package, which is a different but exact delta
because the result's identity is still defined by its own parent edge. A region whose
rebase fails Gate 1R stays on its last good release, or falls back to the trunk; the
dashboard shows which.

**Data and training.**
- Gold scenes need a region tag. Derive it from the capture's device and that device's
  region at capture time, and keep it in the manifest, so a node moving later doesn't
  relabel history.
- The trunk always trains on all regions' data.
- Regional fine-tunes start from the trunk release, use only that region's data, and pin
  the same normalization stats. They don't change the graph, so they diff against the
  trunk.

**Cost to keep in mind.** Every trunk release multiplies the validation and bench work by
the number of regions. Keep the number of regions small, and retire a region (send its
nodes back to the trunk) when its margin over the trunk disappears.

## Edge cases

- **Power loss during download or apply:** staging is resumable and the swap is an atomic
  rename after fsync. On boot, `state.json` resumes or cleans up.
- **Witty Pi shutdown during an update:** the step checks the remaining window first.
  Activation is the only part that must not be cut, and it is a rename plus a reload.
- **Node months behind:** the chain, or the full package if smaller.
- **Node with an unknown or legacy model:** refused; it needs USB provisioning.
- **Node ORT or numpy version differs from the manifest:** the weights still verify by
  sha, but the fingerprint may not match. Report "weights verified, fingerprint mismatch"
  and block confirmation by default. Pin these versions in the provisioning bundle.
- **Disk full:** checked before downloading.
- **Clock wrong:** nothing depends on wall time; signatures don't expire.
- **Two deployments for one node:** one active per node; a newer one supersedes a queued
  one.
- **SIM near exhaustion:** the server refuses above the budget; this could use 1NCE's
  usage API if it becomes available.
- **LoRa-only node:** it shows "awaiting IP"; an update can't go over LoRa.

## Phases

**Phase 0, verify the assumptions.** Run 2026-10-09; the cross-node part ran 2026-10-10.
Complete: every check passed or has a recorded design change.

*Setup.*
- Prototype `wcpkg.py`: tensor-delta-v1, lzma, signed manifest. Base: the fold-0 fp32
  model.
- Packages: full and head-only fine-tunes, each at int8 and int4 (3.13 MB, 1.16 MB,
  273 KB and 82 KB).
- Built in two workstation environments: py3.9 / numpy 1.24.1 / onnx 1.19.1 /
  protobuf 6.33.6, matching the node; and py3.11 / numpy 2.4.6 / onnx 1.17.0 /
  protobuf 7.36.2.
- Applied on 005 (aarch64, py3.9.19, numpy 1.24.1, onnx 1.19.1, protobuf 6.33.6,
  ORT 1.19.2) under `MemoryMax=2800M`, with a peak RSS of 798 MB.

| Check | Result |
|---|---|
| Bit-identical reconstruction | **Pass.** All 4 packages give the same file sha-256 and weights digest on 005 as in both workstation environments, even across protobuf 6 and 7 and numpy 1.x and 2.x. Apply takes 0.13-0.59 s on 005. The integer-only fallback rule isn't needed. Keep `scale` as `np.float32` (numpy 1.x and 2.x promote float64 scalars differently). |
| HF to ONNX mapping | **Partial.** 202/208 map exactly. Export folds the decode head's conv and BatchNorm into 2 computed tensors, so the ONNX file must be canonical (see "Weight spaces"). The other leftovers are the `lo`/`scale` normalization constants, as expected. |
| ORT fingerprint | **Pass: bit-identical across nodes** (re-run 2026-10-10 on 005 and 006).<br>- On each node, logits are identical across runs, fresh sessions and days (005 matched its 2026-10-09 run exactly).<br>- **005 and 006 give bit-identical logits** (max difference 0.0; the same sha at 4 threads and at 1 thread).<br>- Logits differ between 1 and 4 threads, and from x86 by at most 1.8e-6. The argmax mask was identical in every case (0 of 360,448 pixels differ).<br>Rule: record the exact logits sha on the bench Pi with the daemon's 4 threads, and require deployed nodes to match it exactly. Across architectures, use a mask-agreement tolerance (>= 0.9999). |
| Node environment drift | **Found 2026-10-10:** 006's `5band` env has **no `onnx`** module, while 005 has onnx 1.19.1. numpy, ORT and protobuf match. The updater needs onnx to apply a package, so USB provisioning must install pinned versions of onnx (1.19.1 plus `ml_dtypes`) and check them. Nodes should report their package versions in update status, so drift shows on the dashboard. For the test, onnx went into an isolated `pylib_onnx/` on 006 (104 MB).<br>**Resolved 2026-10-10:**<br>- The env file has listed onnx since 2026-09-30 (`9f903f6`). 006's env dates from 2024 and was never rebuilt from it.<br>- onnx 1.19.1 and ml_dtypes 0.5.4 were installed into 006's `5band` env (`--no-deps`; `pip check` clean).<br>- Packages rebuild there to the workstation hashes.<br>- Provisioning should still verify the env against the file. |
| Seeded test input | **Pass.** `default_rng(1234).integers(0, 256, uint8)` gives the same stream on numpy 1.24 aarch64 and numpy 1.24 / 2.4 x86. Numpy doesn't promise this across versions, so pin numpy. |
| Signatures | **Pass.** Pure-Python Ed25519 verifies in 16 ms (RFC 8032 test vector passes, tampered manifest rejected). `cryptography` 50.0.2 installs from wheels in 10 s (18 MB) and verifies in 1.2 ms. Primary path: pure Python, no new dependency. |

The reconstructed model loads and runs in the daemon's configuration on 005 (1.58 s).
Scripts are in the local `session-2026-10-09/phase0/`. Test files are on 005 in
`~/phase0_20261009/`; its `pylib/` is isolated, and the `5band` env wasn't touched.

**Phase 1, format and provisioning:**
- `model_package.py` with tests;
- the node model store;
- daemon `info`/`reload`/`selftest` and default path;
- trial-mode guard;
- release registry;
- USB provisioning bundle;
- provision 005 with `R1`.

**Phase 2, validation:** gates 0-3, the report, the bench-Pi runner, and the Build Release
action in the annotator.

**Phase 3, transport:**
- API tables, endpoints and storage;
- the node `model_update.py` state machine and ticktalk step (with pickle rebuild);
- status reporting;
- the dashboard releases, deploy and progress views.

**Phase 4, rollback and safety:** manual and fleet rollback, revocation, canary gating,
post-deploy monitoring, and the data budget guard.

**Phase 5, smaller updates and extras:**
- head-only and LoRA training modes, with the deterministic LoRA merge;
- automatic encoding search;
- release number in the LoRa status;
- per-node HMAC on status reports.

**Phase 6, regional branches** (only once one region has at least 20 gold scenes from at
least 3 sessions):
- the `model_regions` table and device and neighborhood assignment;
- region tags in the gold manifest;
- the regional training mode;
- Gate 1R;
- the rebase fan-out when a trunk release is approved;
- the regions page and region-aware deploy dialog.

Phases 1-4 already carry the `lineage` manifest field, `releases_on_disk` reporting and
the `base` pointer, so adding branches later needs no node or format change.

## Open questions

- Who holds the signing key and where? The proposal is the annotation workstation, offline
  and passphrase-protected, with the public key on nodes. The server can then only deliver
  releases, never mint them.
- Is head-only or LoRA accurate enough as the default release mode? This can't be judged
  until the gold set is large enough (the current 5-epoch runs used one validation scene).
- How are regions drawn: by geography, by scene type (river, lake, street), or by
  hardware? The plan doesn't care, but Gate 1R's evidence rule makes very fine regions
  impractical.
- Should a regional branch ever be promoted? If a regional head beats the trunk
  everywhere, its data or approach should flow back into the next trunk release rather
  than live on as a branch.
- Should the API also expose a full-model download for a site visit over Wi-Fi?
