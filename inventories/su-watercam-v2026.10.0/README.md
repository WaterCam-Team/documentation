# Node inventories for SU-WaterCam v2026.10.0

CycloneDX SBOMs of the bench units as tested for the release, made on 2026-10-12 with
`python -m tools.node_inventory --mdot-version 4.1.5-mbed60800` (SU-WaterCam
`docs/RELEASES.md`). They list every Debian and Python package, the SU-WaterCam
release, the loaded segmentation model and the Witty Pi and mDot firmware.

| Unit | Release | Model | Components |
|---|---|---|---|
| ufo-01-01-005 | v2026.10.0-dirty (its local no-shutdown edit) | segformer_5band_dyn.onnx `131550d1` | 1185 |
| ufo-01-01-006 | v2026.10.0 | type_pool_b0_dyn_meta.onnx `4ca62324` | 1028 |

**Grype scan (2026-10-12):** no critical or high finding has a fixed version available.
About 10 critical and 40 high matches per unit, in curl, libxml2, python3.13,
util-linux, wget, perl and gcc, are all marked won't-fix or not-fixed by Debian.

They live here rather than on the GitHub release, which is immutable once published,
or in SU-WaterCam, whose files every node downloads on each update.

Scan again with `grype sbom:<file>` when an advisory comes out.
