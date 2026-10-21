# Plan: one shared georeferencing math package

**Written:** 2026-10-20. Not started.

## Why

The same geometry lives in three places, and two of them are copies:

| Where | What |
|---|---|
| Georeferencing `camera_geometry.py`, `geo_core.py` | rotation from heading/pitch/roll; pixel ray; flat-ground intersection; pixel to world; camera elevation from a DEM |
| SU-WaterCam `tools/camera_geometry.py`, `tools/geo_core.py` | identical copies of the two files above (checked 2026-10-20) |
| API `app/georef_math.py`, `app/dem_math.py` | its own `build_rotation_matrix`, `intersect_flat` and `camera_elev_from_dem`, plus EXIF pose reading, heading/pitch/roll resolution, undistortion, intrinsics scaling, ENU to lat/lon, terrain ray-marching, `georeference_pixel(_terrain)` |

A fix in one copy doesn't reach the others, and the API's versions may already
use different conventions (axis order, sign of pitch, which way heading turns).
The mask-to-polygon work and an error budget both need the same functions, so
this is the time to have one copy.

## Shape

- **One package, `ufonet_geo`,** pure Python with numpy only, so it runs on the
  nodes, the API and a laptop. DEM loading (rasterio/GDAL) stays an optional
  extra: `ufonet_geo.dem` imports rasterio lazily; the core never does.
- **Lives in the Georeferencing repo** as `ufonet_geo/` with a `pyproject.toml`,
  versioned by tag (`geo-v1.0.0`). Georeferencing is public; the package holds
  only math, no site data (site data stays in `private/`).
- **Modules:**
  - `pose`: EXIF pose reading; heading, pitch and roll resolution
    (pose_config overrides);
  - `rotation`: `build_rotation_matrix`, with the convention written down;
  - `camera`: intrinsics scaling, undistortion, pixel ray;
  - `ground`: flat intersection, terrain ray-march (DEM as an array plus a
    transform, so it doesn't need rasterio);
  - `geodesy`: local radii, ENU to and from lat/lon;
  - `georef`: `georeference_pixel(_terrain)`, built from the above.

## Consumers

- **API:** depends on the tagged package (`ufonet_geo @ git+…@geo-v1.0.0` in
  requirements; the Docker build has network access). `app/georef_math.py` and
  `app/dem_math.py` become thin wrappers that keep their current names, so routes
  and tests don't change in the first step.
- **SU-WaterCam:** the nodes install from `requirements.txt` over WiFi only and
  shouldn't need GitHub at run time, so they vendor a copy:
  `tools/ufonet_geo/` plus a `VENDORED_VERSION` file, refreshed by a script. A
  CI test fails if the vendored copy differs from the tag it names.
- **Georeferencing tools:** import the package directly.

**Caller check (2026-10-21):** no other repo imports or runs Georeferencing
scripts by path, so moving files there breaks nothing elsewhere. The coupling
is all copies: SU-WaterCam `tools/camera_geometry.py` and `tools/geo_core.py`,
plus a `tools/camera_calibration.py` that has diverged (51-line diff), and the
API's ports in `app/georef_math.py` and `app/dem_math.py`. The one direct file
dependency is API `scripts/seed_ufo006_georef.py`, which reads
`calibration.json` and `unit_config_UFO006.json` from the Georeferencing root;
keep both there (or update that script) when restructuring. Georeferencing's
`LLM_CONTEXT.md` module map now marks each file library / core / helper /
experimental / archived: the *library* rows are what moves into `ufonet_geo`
in step 2. Do any reorganisation of the Georeferencing repo layout as part of
step 5, not as a separate reshuffle first.

## Steps (each a PR)

1. **Golden tests first.** Before moving anything, record outputs of the current
   API functions and the Georeferencing copies for a fixed set of inputs: poses,
   pixels, intrinsics, a small synthetic DEM, and the validation GCPs from
   `private/` reduced to anonymised offsets. Where the two implementations
   disagree, that is a finding: decide which is right (check against the GCP
   validation report) before going further.
2. **Create `ufonet_geo`** in Georeferencing from the API's versions, the more
   complete set, plus anything only the copies have. Port the golden tests and
   add property tests: rotation orthonormal; ray to ground to pixel round trips;
   ENU round trips under 1 mm.
3. **API on the package** through the thin wrappers; golden tests unchanged.
4. **SU-WaterCam vendored copy** replaces `tools/camera_geometry.py` and
   `tools/geo_core.py`; their callers updated; the CI check added.
5. **Georeferencing scripts** import the package; the duplicate files deleted.
6. **Later, on top:** an error budget, propagating pose, intrinsics and DEM
   uncertainty to metres on the ground; and mask-to-polygon in the API, using
   `georef` per boundary pixel and then simplifying.

## Risks and checks

- **A convention mismatch hidden today:** step 1 surfaces it, and it's the main
  value of doing this carefully.
- **The node's Python and numpy versions:** keep the package to the subset both
  the node venv and the API image support; test it on both in CI.
- **Behaviour change on the dashboard:** none intended until step 6. The wrappers
  keep the API's outputs byte-identical to the golden values.

## Decisions needed

- Package home: the Georeferencing repo (proposed), or a new small repo.
- Nodes vendoring a copy (proposed), or installing from a tag over WiFi like
  other pinned packages.
