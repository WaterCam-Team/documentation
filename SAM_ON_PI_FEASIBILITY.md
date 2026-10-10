# SAM2-class segmentation on the Pi 4B: feasibility

Question: can SAM2, or a similar promptable segmentation model, be fine-tuned and
quantized to segment floods on the node, with no human in the loop?

**Short answer:** it would run, but SAM2 is a poor fit as the on-node model. Use it
offline as a label-making teacher for the small model the nodes already run. A tiny
SAM variant on the node is possible only as an optional refinement step.

## What the node has today

- SegFormer-B0 (3.7M params) through ONNX Runtime: **1.5 s per image** on a Pi 4
  (`segformer_5band/docs/ONNX_DEPLOYMENT.md`), all 5 bands, well inside the ~75 s wake.
- The checkpoint is the problem, not the architecture: `iter_100.pth` scores water
  IoU 0.01-0.14. `segformer_5band/NEXT_CHECKPOINT.md` names labels as the limit.
- Prior art: `segformer_5band/tinysam_water.py` picks water points from NIR and thermal
  and prompts TinySAM. The annotator already uses SAM2 for click-to-outline.

## Why SAM2 doesn't fit as the on-device model

1. **No classes.** SAM returns "the object at this prompt", not "water". Unattended use
   needs automatic prompts:
   - grid "segment everything": ~1,000 prompts, mask cleanup, and a separate water
     classifier. Too slow on the Pi.
   - prompts from spectral cues (NIR, thermal, NDWI) or SegFormer's coarse mask:
     workable; this is what `tinysam_water.py` does.
   - fine-tune the decoder with a learned fixed prompt (SAMed / SAM-Adapter style):
     no prompts, but you get a heavy semantic segmenter doing SegFormer's job.
2. **Can't say "nothing here".** SAM always returns a mask, and no-flood frames are the
   common case. Gate prompting on strong spectral evidence and check SAM's mask
   against the spectral mask.
3. **RGB only.** NIR and thermal can only enter through the prompts, unless the patch
   embedding is changed and the pretraining that makes SAM worth using is lost.
4. **SAM2's video memory doesn't help.** Captures are hours apart with power-off in
   between, so it is effectively a SAM1-style image model.

## Quantization limits on the Cortex-A72

- **fp16: no speedup.** ARMv8.0 has no fp16 arithmetic; only storage shrinks.
- **int8: measured 1.0-1.3x, not 4x** (no dot-product instructions; LayerNorm, softmax,
  GELU stay fp32). See the measurements below.
- SAM encoders are sensitive to post-training quantization: calibrate on real captures,
  keep the decoder fp32, and check water IoU after quantizing.

## Power

Sustained 4-core inference is the load that browned out 006 at 2000 MHz
(`SU-WaterCam/docs/POWER_ANALYSIS.md`: 8-10 W peaks). At stock clocks, ~10 s of extra
inference at ~6 W is ~0.017 Wh per image; at 2 captures x 9 wakes that is ~0.3 Wh/day,
about 10% of the 2.5-3.5 Wh/day budget.

## Fine-tuning

Easy and off-device: freeze the encoder, train the ~4M-param mask decoder (optionally
LoRA on the encoder) on the lab GPU. A few hundred session-split masks is enough for a
decoder. Compute isn't the bottleneck once labels exist.

## Recommendation

1. **Main path:** SAM2-L offline (already in the annotator) labels captures in bulk;
   people verify; export with `--gold-only`; retrain SegFormer-B0/B1 (with the 3- vs
   5-band ablation); ship through the existing ONNX daemon, int8-quantized. Keeps the
   1.5 s budget, all five bands, semantic output, correct empty masks, no prompting.
2. **Optional refinement on the node:** EdgeSAM or TinySAM, run only when SegFormer and
   NDWI agree water is present, prompted from their mask. Adopt only if it raises water
   IoU on the gold masks enough to pay for its time and energy (see measurements).
3. **Not recommended:** SAM2 Hiera-T or larger as the only on-node model.

`tinysam_water.py` against the gold masks: see the last section. The segmenter is good
when prompted well; the unattended prompt picker is what fails.

## Measurements on 005 (2026-10-08)

**Setup.** Pi 4B Rev 1.5, 4 GB, wall-powered. CPU capped at stock 1.8 GHz through cpufreq
(005 is normally at `arm_freq=2000`; the cap was restored afterwards). ONNX Runtime 1.19.2
on the CPU provider, 4 intra-op threads, all graph optimisations, in the `5band` env. Each
case ran in its own process: 1 warm-up run, then median of 5 (3 for SAM2). Encoders got
one real capture (1296x972 NIR-OFF, long side resized to 1024, padded to 1024² except
MobileSAM, whose graph takes 768x1024 HWC and does its own preprocessing). SegFormer got
random 1x5x512x704, the shape the daemon pads to. Decoders got the encoder's output plus a
fixed 5-point prompt and stayed fp32. Current was sampled at about 2 Hz from the Witty Pi
output (5 V side), so short peaks can be missed.

**int8 variants.** "int8" means ORT dynamic quantization of MatMul/Gemm weights. "QDQ"
means static per-channel QDQ, calibrated on 8 captures. Agreement is measured against the
fp32 model on the same input: embedding cosine, and IoU of the decoder's masks.

| Model | Encoder per image | Decoder per prompt | Load | Peak RSS | Mean / peak current | Agreement with fp32 |
|---|---|---|---|---|---|---|
| SegFormer-B0 5-band fp32 (deployed) | **1.62 s** | — | 0.3 s | 490 MB | 1.14 / 1.54 A | — |
| SegFormer-B0 int8 | 1.53 s | — | 0.5 s | 450 MB | 1.17 / 1.39 A | logits cos 0.995, 99.5% pixels agree |
| EdgeSAM (3x) fp32 | **2.87 s** | 0.34 s | 0.6 s | 375 MB | 1.21 / 1.41 A | — |
| EdgeSAM int8 (dynamic) | 2.80 s | 0.37 s | 0.6 s | 383 MB | 1.19 / 1.37 A | cos 1.000 (no MatMuls to quantize) |
| EdgeSAM QDQ (static) | 2.73 s | 0.34 s | 0.5 s | 376 MB | 1.13 / 1.46 A | **cos 0.25, mask IoU 0.52: broken** |
| MobileSAM / TinySAM-class fp32 | 5.55 s | — | 1.1 s | 562 MB | 1.19 / 1.44 A | — |
| MobileSAM int8 | 4.76 s | — | 0.9 s | 550 MB | 1.21 / 1.54 A | cos 0.996 |
| SAM2 Hiera-T fp32 | **13.6 s** | 0.40 s | 3.6 s | 1,604 MB | 1.26 / 1.54 A | — |
| SAM2 Hiera-T int8 | 10.9 s | 0.39 s | 1.7 s | 1,628 MB | 1.24 / 1.48 A | cos 0.98-0.997, **mask IoU 0.88** |
| SAM2 Hiera-S fp32 | 16.9 s | — | 4.3 s | 1,593 MB | 1.27 / 1.59 A | — |
| SAM2 Hiera-S int8 | 12.7 s | — | 2.0 s | 1,541 MB | 1.25 / 1.67 A | cos 0.98-0.997 |

Idle was 0.48 A at 4.88 V (2.3 W). During inference the draw was 5.4-6.1 W on average,
with peaks of about 1.7 A (8 W). There was no throttling (`get_throttled` 0x0 throughout).
SoC temperature rose from about 56 °C to 78 °C over the SAM2 runs on the bench, close to
the 80 °C soft-throttle point, so a sun-heated enclosure would throttle long SAM2 runs.

**What changed from the estimates**

- **int8 buys little: 1.0-1.3x.**
  - SegFormer: 6%.
  - EdgeSAM: nothing from dynamic quantization, because it is conv-based (RepViT).
    Uncalibrated-style static QDQ destroys it; it would need QAT or a careful mixed-precision
    pass.
  - SAM2-T: 20% faster, at 0.88 mask IoU against its own fp32 output.
- **Decoders cost 0.34-0.40 s per prompt**, not the tens of ms estimated. Grid "segment
  everything" (~1,000 prompts) would take several minutes, which rules it out.
- **Energy per image above idle** is (P - 2.3 W) x time, so the first wake also pays load
  plus warm-up:

  | Model | Energy per image | Per day (2 captures x 9 wakes) |
  |---|---|---|
  | SegFormer-B0 | ~5 J (0.0014 Wh) | ~0.03 Wh |
  | EdgeSAM, encoder + 1 prompt | ~11 J (0.003 Wh) | ~0.06 Wh |
  | SAM2-T fp32 | ~52 J (0.014 Wh) | ~0.26 Wh |
  | SAM2-T int8 | ~41 J | ~0.20 Wh |

  SAM2-T at 0.20-0.26 Wh/day is 7-10% of the 2.5-3.5 Wh/day budget.

**Verdict, confirmed.** SAM2-T/S at 11-17 s, 1.6 GB and up to 8 W peaks is not worth it on
the node. EdgeSAM at about 3.2 s (fp32 encoder plus one prompt) is the only SAM-class model
cheap enough for an optional, gated refinement stage, and it should stay fp32. Quantizing
the deployed SegFormer to int8 isn't worth it either (6%).

**Caveats.**
- One image, 3-5 runs, `ondemand` governor, bench temperature.
- The SAM2 ONNX exports are third-party (`vietanhdev/segment-anything-2-onnx-models`,
  SAM2.0 rather than 2.1). EdgeSAM and MobileSAM are from `chongzhou/EdgeSAM` and
  `Acly/MobileSAM` on Hugging Face.
- Agreement is measured against fp32, not against gold masks. Water IoU still needs
  measuring.

**Incident during the test.**
- **005 rebooted** while ORT static calibration (`quantize_static`, EdgeSAM, 8 images) ran
  on the node. The Witty Pi reason was `0x0b` (reboot; power was not lost), with no
  undervoltage bits.
- The same calibration on the workstation peaked at **4.4 GB RSS**, more than the Pi's
  3.8 GB. The likely sequence: memory ran out, the system froze, and systemd's 1-minute
  hardware watchdog reset it. The RAM journal from that boot is lost, so this is inferred.
- Static calibration has to run off the node. The benchmark itself ran in a cgroup
  (`MemoryMax=2800M`, `MemorySwapMax=0`, `OOMPolicy=continue`) with no further incident.

Scripts, models and raw data are on 005 in `~/sam_bench_20261008/` (`bench.py`,
`quantize.py`, `run.sh`, `models/`, `results/results.jsonl`, `results/power.csv`).

## `tinysam_water.py` against the gold masks (2026-10-08)

**Setup.**
- Gold set: the two hand-labelled scenes in `photo_processing/example_data` (Brooklyn,
  Onondaga). These are the same masks the deployed SegFormer was scored on (water IoU
  0.14 and 0.01).
- Masks were read with `training.data.read_mask` and scored with `training.metrics.Metrics`
  (water IoU, precision and recall).
- The script's `run()` was used unchanged, on the workstation CPU with torch 2.5.1. TinySAM
  is commit `11589bc`, with release-3.0 weights `tinysam.pth` and `tinysam_42.3.pth`.
- TinySAM's mask comes out on the 2592x1944 NIR-OFF JPEG grid. It was downscaled (nearest
  neighbour) to the 1296x972 TIFF and gold grid. The two grids are aligned: the downscaled
  JPEG matches the TIFF's RGB bands at correlation 0.999 with zero shift.

**Bug found.** TinySAM's builder calls `torch.load` without `map_location`, and its
checkpoints hold CUDA tensors. The script therefore fails on any CPU-only machine, the Pi
included ("Attempting to deserialize object on a CUDA device"). The scoring harness
defaulted `torch.load` to CPU; the script still needs that fix before it can run on a node.

**Results** (water IoU; P = precision, R = recall):

| Scene (gold water %) | Spectral candidates alone | TinySAM, 5 points (default) | TinySAM, 1 point | TinySAM, 5 points placed in gold water |
|---|---|---|---|---|
| Brooklyn (52%) | 0.16 (P 0.57, R 0.19) | **0.90** (`42.3` weights: 0.89) | 0.00 | 0.998 |
| Onondaga (40%) | 0.19 (P 0.47, R 0.24) | **0.00** | 0.00 | 0.946 |

Both weight files gave the same picture. The "placed in gold water" column is a ceiling,
not something achievable unattended: 5 draws of points inside the eroded gold mask,
median shown.

**Why it fails: the prompt picker treats the sky as water.**
- `pick_water_points_from_bands` keeps pixels below the per-image median in both the NIR
  difference band and thermal. Sky passes that rule: it is cold in thermal and has little
  NIR difference.
- The picker then prompts the largest blobs first, and the sky is usually the largest.
- In Onondaga, all 5 prompts land in the sky, the trees or the far shoreline. TinySAM
  outlines that strip, giving 14% predicted water with zero overlap with the gold mask.
- In Brooklyn with 1 point, the prompt is in the sky and TinySAM returns the sky. With 5
  points, 4 land on the river and TinySAM returns the whole river plus a cloud patch.
- The median rule also misses bright reflections on water. The spectral candidates cover
  only 19-24% of the gold water.

**Takeaways.**
- **The segmenter is not the problem.** With well-placed prompts, TinySAM scores
  0.95-0.998, far above the deployed SegFormer (0.14 / 0.01).
- **One point is unreliable** even when placed correctly. On choppy water it often outlines
  a single wave (Brooklyn median 0.001). Keep 5 or more points.
- **The prompt picker needs to exclude the sky** before this is usable unattended. Options:
  - drop candidate blobs above the horizon (from the IMU pitch, or the image's horizon line);
  - drop blobs touching the top border;
  - use SegFormer's mask as the prompt source once a trained checkpoint exists.
  Any such rule is tuned against these same two scenes, so it needs validating on new gold
  masks.
- **Two scenes is far too few to rank models.** Treat these numbers as a diagnosis, not a
  benchmark.

Harness, overlays and bench copies are local only, in the workspace folder
`session-2026-10-08/` (`score_tinysam.py`, `tinysam_scores/overview.jpg`, `sambench/`, `sam_bench_005_results/`). They are not in any repo.
