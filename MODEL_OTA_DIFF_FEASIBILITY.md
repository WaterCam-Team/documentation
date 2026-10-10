# Updating the node's segmentation model by diff: feasibility

Question: can a deployed node's SegFormer model be updated by sending a diff against the
model it already has, to keep transfers small?

**Short answer: yes, but not with a generic binary diff.** `xdelta3` or
`zstd --patch-from` on the ONNX file saves nothing for any update that retrains the
whole network. What works is a **tensor-level, quantized weight delta**:
- the server reconstructs exactly what the node will reconstruct;
- it validates that reconstruction on the gold masks;
- the node rebuilds the file and checks its SHA-256 before swapping it in.

**How small an update can be depends on how the update is trained, more than on the diff
tool.**

| Update | Size | Share of a SIM's 500 MB lifetime cap |
|---|---|---|
| Full fine-tune, int4 delta | ~1.2 MB | ~0.25% |
| Head-only fine-tune, int8 delta | ~290 KB | ~0.06% |
| Head-only fine-tune, int4 delta | ~80 KB | ~0.02% |
| Full model compressed (no diff) | ~13.7 MB | ~2.7% |

## Constraints

- **Data:** 1NCE SIMs have a **500 MB lifetime** cap (`SU-WaterCam/docs/1NCE_SIM_EXHAUSTION.md`).
  Tailscale already costs 126-474 KB per wake.
- **Link:** Quectel EC25-AF, LTE Cat 4. Even a full model downloads in seconds to a couple of
  minutes, so airtime and the awake window aren't the limit; bytes against the cap are.
- **What nodes run:** fp32 ONNX. INT8 (static QDQ) cost 0.667 water IoU for about 15% speed,
  so fp32 stays (`WORK_SUMMARY_2026-09-23_to_2026-10-07.txt`). The diff therefore has to
  produce an fp32 file.
- **The model file:** `segformer_5band_fp32.onnx` is 15.14 MB (3.72M params). Of that,
  396k params are the decode head; the rest is the MiT-B0 encoder, initialised from
  ImageNet (`nvidia/mit-b0`).

## Experiment (workstation, 2026-10-09)

**Base model.** CV fold 0 of the five-band SegFormer-B0
(`photo_processing/annotator/work/compare/cv-fiveband__segformer-b0-0/best_hf`).

**Updates applied to it,** all with `training.train`, 5 epochs, lr 6e-5, normalisation
pinned with `--stats` to the base's `norm.json`:
- **full fine-tune** on all labelled scenes;
- **head-only fine-tune,** same run with every `segformer.*` (encoder) parameter frozen:
  395k of 3.72M trainable;
- **independent retrain:** CV fold 1, trained separately from the same ImageNet init.
  This is the worst case.

**Export and scoring.** Every model was exported with the node bundle builder
(`annotator/trainer.py --export-pi`). Export is deterministic: re-exporting the same
checkpoint gives byte-identical fp32 and int8 files, so diffs measure weight changes only.
"Agreement" is argmax agreement between the reconstructed model and the exact new model
on 6 real five-band captures. It is not IoU against gold masks.

### fp32 file, the one nodes run (15.14 MB raw)

| Old -> new | Compressed full file (xz -9e) | `xdelta3` / `zstd --patch-from` | Exact tensor delta (XOR) | Lossy int8 delta | Lossy int4 delta |
|---|---|---|---|---|---|
| Full fine-tune (100% of weights changed) | 13.74 MB | 13.81 / 13.80 MB | 11.66 MB | **3.23 MB**, water IoU 0.9999 | **1.21 MB**, water IoU 0.998 |
| Head-only (12/206 tensors, 10.6% of weights) | 13.74 MB | 1.46 / 1.47 MB | 1.32 MB | **288 KB**, water IoU 0.9998 | **82 KB**, water IoU 0.993 |
| Independent retrain (fold 1) | 13.75 MB | 13.81 / 13.79 MB | 11.88 MB | 3.06 MB, water IoU 0.9994 | 1.07 MB, water IoU 0.953 |
| Currently deployed mmseg `iter_100` -> new pipeline | — | 13.81 MB | — (different graph) | — | — |

**Baseline without a diff.** The whole new model quantized the same way (per-tensor,
weight-only) is 2.73 MB at int8 with water IoU 0.982, and 0.81 MB at int4 with water IoU
**0.0** (it collapses).
- At int8 the delta is about the same size but far more faithful (0.9999 against 0.982).
- At int4 only the delta survives.
- So the diff earns its place at int4, and for partial updates.

### int8 file, for reference

These are exact, lossless deltas: the int8 values as (new - old) mod 256, then zstd.

| Update | Raw | zstd full file | `zstd --patch-from` | Tensor delta |
|---|---|---|---|---|
| Full fine-tune | 4.64 MB | 3.54 MB | 1.70 MB | **665 KB** |
| Head-only | 4.64 MB | 3.54 MB | 290 KB | **98 KB** |
| Independent retrain | 4.64 MB | 3.54 MB | 2.12 MB | 775 KB |

Small int8 fine-tunes leave most int8 values unchanged (27% changed for the full
fine-tune), so exact int8 deltas are cheap. That is only useful if the int8 accuracy
problem is fixed.

### Low-rank updates (sizes only; accuracy not measured)

Counted from the real layer shapes (48 encoder linear layers, 2.31M params):

| Update | Params | Size |
|---|---|---|
| LoRA rank 4 | 74k | 295 KB fp32 / 74 KB int8 |
| LoRA rank 8 | 147k | 590 KB fp32 / 147 KB int8 |
| LoRA rank 16 | 295k | 1.18 MB fp32 / 295 KB int8 |
| Decode head (shipped alongside) | 396k | 1.58 MB fp32 / 396 KB int8; a head delta is smaller still (above) |

LoRA would also need merging into the fp32 weights on the node, or a graph change.

## Why the generic binary diff fails

Fine-tuning at even lr 6e-5 for 5 epochs changes **every** fp32 weight, by tiny amounts
spread across the mantissa bits. Byte-level diff tools see an entirely new file. Exact
XOR deltas only recover the sign and exponent bits that stayed the same, which saves
about 15%. The savings come from deciding how much precision the delta needs, and that
decision has to be validated.

## Requirements

1. **The first update cannot be a diff.** The deployed mmseg `iter_100` model shares
   neither graph nor weights with the new training pipeline (patch 13.8 MB, no gain). Ship
   one full model, about 13.7 MB compressed (2.7% of the lifetime cap per node), or load
   it at a site visit over Wi-Fi or SD. Diffs only pay from the second update on.
2. **The server must know each node's exact base.** The node should report its model's
   SHA-256 (in its readings or `/health`). The server keeps every released version and
   builds the delta for each (from, to) pair a node actually needs. A node that missed a
   release gets a delta from its own version, not a chain.
3. **The graph must stay identical between versions.** It was identical for both
   fine-tunes, and differed for fold 1, whose normalisation constants are compiled into
   the graph. Pin `--stats` across releases so only initializers change. Otherwise ship
   the (small) graph separately.
4. **Validate the reconstructed model, not the trained one.**
   - The release step should apply the quantized delta to the base, score the result on
     the gold masks, and record its SHA-256 in the manifest.
   - The node rebuilds the file, checks the hash, runs a smoke test on a stored capture,
     and swaps atomically. It keeps the previous model on disk for rollback, then
     restarts `segformer_daemon`.
5. **Reconstruction must be bit-identical on x86 and ARM.** The format uses per-tensor
   `old + q * scale` in float32 numpy, which should be deterministic (IEEE single, no
   fused multiply-add in numpy elementwise operations). **This has not been checked on a
   Pi.** The hash check catches a mismatch, and if it ever fails, reconstruction can move
   to integer arithmetic.
6. **Transport.** A checksummed download that can resume (HTTP Range) costs little. Over
   Cat 4, a 1-3 MB delta finishes inside one wake. LoRa is out of the question.

## Recommendation

- Treat **"updates are head-only or low-rank, shipped as an int8 delta"** as the default
  release shape: about 0.1-0.3 MB, roughly the cost of one wake's Tailscale traffic.
- Keep **full fine-tune plus int4 delta** (about 1.2 MB) for releases that need the encoder
  to move.
- Keep **full model transfer** for the first rollout and for architecture changes.

Before this is usable:
- Test bit-identical reconstruction on 005.
- Measure head-only and LoRA against full fine-tuning on the gold masks once there are
  enough of them. The validation scores from these 5-epoch runs use one validation scene
  and say nothing about which strategy is more accurate.

## Caveats

- Only one fine-tune of each kind, 5 epochs on CPU. Longer or larger fine-tunes move
  weights further, so their deltas are larger and less faithful at int4 (the fold 1
  retrain drops to 0.953).
- The quantizer is per-tensor symmetric. A per-channel version would be more faithful at
  the same size.
- Agreement is measured on 6 captures against the exact model, not against gold labels.

Scripts and data (local only, not in any repo): `session-2026-10-09/`.
