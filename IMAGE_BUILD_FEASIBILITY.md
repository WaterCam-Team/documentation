# Custom Pi 4 node image: feasibility

Question: can pi-gen, Buildroot or Yocto build a Raspberry Pi 4 image that already
contains SU-WaterCam, the 5-band SegFormer daemon, the Witty Pi software, their
dependencies and everyday tools (rsync, wget, htop)? Can the hostname, Tailscale
install and Tailscale SSH be set up automatically?

**Short answer:** yes, with **pi-gen**. Buildroot is a poor fit. Yocto would work but
costs far more than it returns for a team our size. Automatic hostnames
(`ufo-01-01-NNN`), Tailscale install and Tailscale SSH can all be done at first boot.
The one catch is that the Tailscale auth key is supplied per card at flash time, never
built into the image.

## How nodes are set up today

Nothing is automated:

- Nodes are flashed from `dd` + PiShrink clones of a golden card, stored in
  `sd-images` (Git LFS).
- After flashing, these are all done by hand from `SU-WaterCam/README.md` and
  `SU-WaterCam/docs/FIELD_DEPLOYMENT_GUIDE.md`:
  - the hostname and the `/etc/hosts` fix;
  - Tailscale install and `tailscale up --ssh`;
  - copying and enabling the systemd units;
  - the Witty Pi scripts and schedule;
  - `runtime_config.json`.
- The base OS is Raspberry Pi OS Lite 64-bit, Trixie (Debian 13). Nodes no longer use
  Bookworm; 005 and 006 are verified on Trixie (`SU-WaterCam/requirements.txt`).
- Clones copy the machine-id, the NetworkManager secret key and the DHCP DUID onto
  every node. FIELD_DEPLOYMENT_GUIDE §1.2 lists the steps to clear them.
- The segformer daemon runs in a separate conda env (`miniforge3/envs/5band`: Python
  3.9, onnxruntime 1.19.2, torch 1.7.1). `tools/segformer_daemon.py` only imports
  `onnxruntime`, `cv2` and `rasterio`.

## The three options

| | pi-gen | Buildroot | Yocto |
|---|---|---|---|
| Result | Raspberry Pi OS Lite (trixie, arm64), the OS nodes already run | Minimal custom system, no package manager | Custom distro on the meta-raspberrypi layer |
| picamera2, libcamera, ov5647, rpicam-apps | Work as is, from apt | Hard: the Pi's libcamera fork and picamera2 must be packaged by hand | Recipes exist; integration work needed |
| GDAL/rasterio, pandas, scipy, OpenCV | apt | Possible but painful | meta-oe and meta-python have them |
| onnxruntime | pip wheel | No package; cross-build it | Community layer only |
| Field workflow (`git pull`, `apt install`, debugging over SSH) | Unchanged | Mostly lost | Mostly lost unless package feeds are added |
| Build cost | About 1 h in Docker, small config | Long builds, every package declared | 50–100 GB disk, hours per build, steep learning curve |
| Advantage over pi-gen | — | Small image, fast boot | Reproducible builds; A/B over-the-air updates (Mender, RAUC, SWUpdate) |

- **Buildroot** suits sealed appliances. Our nodes are research machines that people
  log into and change, and the software depends on Pi-specific camera and GIS Python
  packages that Buildroot doesn't provide.
- **Yocto** pays off only if we move to whole-system A/B updates across a large fleet.
  Revisit it then, alongside `MODEL_UPDATE_SYSTEM_PLAN.md`.

## pi-gen build plan

pi-gen is the tool Raspberry Pi uses to build Raspberry Pi OS. We would keep its
stages 0–2 (Lite) and add one custom stage, `stage-watercam`, before the export step.

The stage would:

1. **Install apt packages:**
   - the list in the `requirements.txt` header;
   - rpicam-apps, gpsd, ModemManager, NetworkManager, doas, i2c-tools, build-essential,
     git, tmux;
   - rsync, wget, htop;
   - `tailscale`, from the Tailscale apt repo for trixie.
2. **Install the software:**
   - clone SU-WaterCam and segformer_5band at pinned commits into `/home/pi`;
   - create the venv with `--system-site-packages` and install the pinned pip packages;
   - compile the Lepton tools (`tools/lepton.c`, `tools/capture.c`);
   - fetch the ONNX model (see below).
3. **Copy config from `SU-WaterCam/config/`:**
   - `config.txt` and `cmdline.txt`;
   - every systemd unit, enabled;
   - `doas.conf`, the polkit NetworkManager rule, the journald drop-in, the Quectel
     udev rule;
   - a NetworkManager profile for the "Quectel" cellular connection;
   - the Witty Pi scripts (`beforeScript.sh`, `syncTime.sh`, `gps_time.py`) and the
     `.wpi` schedules.
4. **Apply system settings:** ext4 `noatime,commit=60`, tmpfs `/tmp`, no swap
   (`dphys-swapfile` removed).
5. **Set pi-gen options:** user `pi` (`FIRST_USER_NAME`), skip the first-boot rename
   wizard, enable SSH.

### Things to watch

- **Witty Pi installer.** Don't run UUGear's `install.sh` in the build: it downloads
  at run time and edits files in place. Reproduce its effects instead:
  - add `i2c-dev` to `/etc/modules`;
  - set the I2C dtparams in `config.txt`;
  - install the `/etc/init.d/wittypi` script and the Witty Pi software into
    `/home/pi/wittypi`;
  - leave out `uwi`.
- **Drop the conda env.** The daemon needs only onnxruntime, OpenCV, rasterio and
  Pillow. It can run in the SU-WaterCam venv: rasterio and numpy come from apt, OpenCV
  4.13 is already pip-installed there, and onnxruntime is one more pip wheel. That
  removes miniforge, Python 3.9 and torch 1.7, which are the largest and most fragile
  part of the image. `segformer_daemon.service` would then point at the venv's Python.
  Tested on 005 (2026-10-11); see the results below.
- **Model source.** `segformer_5band_int8.onnx` (4.6 MB) currently exists only in
  `photo-masks/annotator/weights/`. It needs a canonical home, such as a GitHub release
  asset, that the build can download by version.
- **Build host.** pi-gen runs under Debian or in Docker (`build-docker.sh`, which needs
  a privileged container and qemu-user-static on non-arm64 hosts). pip builds under
  emulation (RPi.GPIO, for example) are slow. A GitHub `ubuntu-24.04-arm` runner or a
  Pi 5 builds natively.
- **Clone problems go away.** A built image ships with an empty machine-id, and
  Raspberry Pi OS generates SSH host keys on first boot. The §1.2 cleanup steps are no
  longer needed.
- **Storage.** A compressed image in `sd-images` must stay under the 2 GB LFS file
  limit. The current minimal image is 1.9 GB; dropping conda helps. Better: keep the
  pi-gen config in git and build images on demand, or in CI, instead of storing every
  image.
- **Where the config lives.** The stage holds no secrets, so it can go in a public repo
  (a new `image/` directory in SU-WaterCam or its own repo). Per-node secrets are
  supplied at flash time.

## Hostname and Tailscale at first boot

Build one generic image. After flashing, a small script writes two files to each
card's FAT boot partition (`/boot/firmware`):

- `ufo-node-id`: the node number, `NNN`;
- `ts-authkey`: a Tailscale auth key.

A one-shot `watercam-firstboot.service` (which runs only while `ufo-node-id` exists)
then:

1. sets the hostname to `ufo-01-01-NNN` and fixes the `127.0.1.1` line in `/etc/hosts`;
2. writes `device_id` into `runtime_config.json`;
3. waits for the network and runs:
   ```
   tailscale up --auth-key=file:/boot/firmware/ts-authkey --ssh \
     --hostname=ufo-01-01-NNN --advertise-tags=tag:watercam
   ```
4. deletes `ts-authkey` and `ufo-node-id`, then disables itself.

Notes:

- **NNN can't come from the hardware.** It is a number we assign. The alternative is a
  table in the image mapping Pi serial numbers to node numbers, which still has to be
  filled in before flashing. The prompt in the flashing script is simpler.
- **cloud-init.** Trixie Raspberry Pi OS images may also accept a cloud-init
  `user-data` file on the boot partition for the hostname. The custom service works
  either way and also handles `runtime_config.json` and Tailscale.
- **Auth key type.** Use a single-use, pre-approved key tagged `tag:watercam`. Tagged
  devices have key expiry off by default, and the existing ACL
  (`SU-WaterCam/config/tailscale-acl.txt`) already allows SSH as `pi` to that tag.
  - Better: the flashing script mints a fresh key per card through a Tailscale OAuth
    client (auth-keys scope), so no long-lived reusable key ever sits on a card.
  - Never build a key into the image. Keys must not be shared between cards, and the
    image ends up in `sd-images`.
- **Network at first boot.** The node needs a connection on first boot: bench Wi-Fi, or
  the cellular modem through the baked-in Quectel profile. The service retries until
  `tailscale up` succeeds.
- **Reflashing an existing node.** Delete its old entry in the Tailscale admin console
  first. Otherwise the node comes back as `ufo-01-01-NNN-1`.

## What stays manual

These are per-node secrets or hardware state, not image content:

- the API key in `runtime_config.json`. The flashing script can prompt for it with NNN
  and write it to the boot partition the same way;
- the mDot DevEUI, set over AT commands and registered in ChirpStack;
- choosing the Witty Pi schedule and saving or restoring the RTC offset;
- flashing Witty Pi firmware.

## Test: segformer daemon without conda (005, 2026-10-11)

Setup:

- The same 20 captures (`color_preserved_5_band.tiff`, April–October 2026) were run
  through `segformer_daemon.run_inference` in two environments.
- Conda env: Python 3.9.19, onnxruntime 1.19.2, OpenCV 4.5.1, numpy 1.24.
- SU-WaterCam venv: Python 3.13.5, onnxruntime 1.31.0 (temporary pip install),
  OpenCV 4.13.0, numpy 2.2.4.
- Each run was capped at 2.8 GB with `systemd-run`.

| Model | Mask pixels that differ | Max logit difference | Median inference time |
|---|---|---|---|
| fp32 deployed on 005 (`segformer_5band_dyn.onnx`) | 2 of 25.2 M | 5e-6 | 1476 → 1327 ms |
| int8 (`segformer_5band_int8.onnx`, from photo-masks) | 0.27% | 0.6 | 910 → 801 ms |

- **fp32:** the masks are identical to within float rounding.
- **int8:** the preprocessed inputs were bit-identical in both environments, so the
  difference comes from onnxruntime's quantized kernels changing between versions, not
  from OpenCV or numpy.
  - Against the matching fp32 model, run on the same inputs, int8 on 1.19.2 disagreed
    on 0.044% of model-resolution pixels and int8 on 1.31.0 on 0.039%. The newer
    runtime is no worse.
- The newer stack is about 10% faster on both models.
- onnxruntime 1.31.0 imports and runs without its optional dependencies (installed
  with `--no-deps`).

Notes:

- 005 still runs the older fp32 model. The int8 model isn't deployed there.
- System Python on 005 has no `cv2`; the venv's pip OpenCV provides it. The image
  should do the same, or install apt's `python3-opencv` and drop the pip copy.

## Effort and next steps

About 2–4 days to a first working pi-gen image, plus a bench check against the current
setup.

1. ~~Test the segformer daemon without conda on 005.~~ Done 2026-10-11: it works (see
   above). Add a pinned onnxruntime to `requirements.txt`, point
   `segformer_daemon.service` at the venv, and run it on 005 for a while before
   removing conda.
2. Publish the int8 model as a versioned release asset.
3. Write the pi-gen stage and the first-boot service; build in Docker or on an arm64
   runner.
4. Write the flashing script (asks for NNN and the API key, mints the Tailscale key,
   writes the boot-partition files).
5. Flash a spare card for 006 and compare services, packages and captures with the
   current image.
6. Update `sd-images/README.md` and FIELD_DEPLOYMENT_GUIDE to the new flow.
