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
   - `systemd-resolved` (it pulls in `libnss-resolve`), so NetworkManager and Tailscale
     share DNS instead of overwriting each other's `/etc/resolv.conf` (see below);
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
   - the NetworkManager drop-in `config/networkmanager/90-dns-systemd-resolved.conf`
     (SU-WaterCam #145), with `/etc/resolv.conf` a symlink to
     `../run/systemd/resolve/stub-resolv.conf`;
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
  Pillow. It runs in the SU-WaterCam venv: rasterio, numpy and OpenCV
  (`python3-opencv`) come from apt, and onnxruntime is one more pip wheel. That
  removes miniforge, Python 3.9 and torch 1.7, which are the largest and most fragile
  part of the image. `segformer_daemon.service` already points at the venv's Python
  (SU-WaterCam #142); see the test results below.
- **systemd-resolved in the chroot.** Once `/etc/resolv.conf` points at resolved's
  stub, DNS inside the build chroot stops working, because resolved isn't running
  there. Install it and make the symlink as the stage's last step, after every
  download.
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

## DNS: systemd-resolved

Raspberry Pi OS Lite doesn't install systemd-resolved, so NetworkManager writes
`/etc/resolv.conf` directly, and so does Tailscale when MagicDNS is on.

**What goes wrong without it:**

- NetworkManager rewrites the file each time a connection comes up or goes down,
  including the Quectel connection the IP uplink starts on demand.
- Each rewrite drops Tailscale's `100.100.100.100`. Tailnet names, including the API
  server's, stop resolving until tailscaled writes the file again.
- On 006 (2026-10-11) that took 26 s in one test, and in another the name still didn't
  resolve 40 s after Quectel came up.
- The failure is immediate: `curl` reports `Could not resolve host`, not a timeout.
  The cycle then decides the server is unreachable over WiFi and brings Quectel up
  again. 006 did that on every wake, and it spends cellular data even with WiFi up.

**With systemd-resolved:**

- NetworkManager gives resolved each connection's DNS servers.
- Tailscale gives it `100.100.100.100` for the tailnet's domains only (split DNS).
- Nothing overwrites anything.

**Tested on 006 (2026-10-11):**

| Test | Before | With resolved |
|---|---|---|
| Quectel up, then down, `/health` by name every second | 26 s to never-recovered of failures | no failures |
| Wake cycle with WiFi up | server unreachable twice, Quectel brought up twice | both uplinks over WiFi, Quectel untouched |
| Wake cycle with WiFi down | not run | Quectel up once, both uplinks sent, no flapping |
| Reboot | not run | resolved, NetworkManager and Tailscale come up; server resolves at once |

005 was switched the same way that day. Both nodes keep the old files in
`/root/dns-before-resolved-20261011`.

**On an existing node** (the steps are in SU-WaterCam's FIELD_DEPLOYMENT_GUIDE, section
1.3):

1. `apt install systemd-resolved`. The package points `/etc/resolv.conf` at the stub.
2. Add the NetworkManager drop-in above and reload NetworkManager.
3. Restart tailscaled. It picks its DNS mode when it starts.

Restarting tailscaled drops a Tailscale SSH session, so run the steps detached, for
example under `systemd-run`, with a rollback if the server name doesn't resolve
afterwards. A built image needs none of this: resolved is in place before Tailscale
first starts.

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
- At the time, system Python on 005 had no `cv2` and the venv's pip OpenCV provided it.
  005 has since moved to apt's `python3-opencv` 4.10, like 006, and the segmentation
  output was unchanged. The image should install `python3-opencv` and no pip copy.

## Effort and next steps

About 2–4 days to a first working pi-gen image, plus a bench check against the current
setup.

1. ~~Test the segformer daemon without conda on 005.~~ Done 2026-10-11 (see above).
   SU-WaterCam #142 pins onnxruntime in `requirements.txt` and points
   `segformer_daemon.service` at the venv. 005 and 006 both run it that way, with
   apt's OpenCV. Conda stays until the venv has run for a while.
2. ~~Add the systemd-resolved NetworkManager drop-in to `SU-WaterCam/config/` and to
   the setup guide.~~ SU-WaterCam #145.
3. Publish the int8 model as a versioned release asset.
4. Write the pi-gen stage and the first-boot service; build in Docker or on an arm64
   runner.
5. Write the flashing script (asks for NNN and the API key, mints the Tailscale key,
   writes the boot-partition files).
6. Flash a spare card for 006 and compare services, packages and captures with the
   current image.
7. Update `sd-images/README.md` and FIELD_DEPLOYMENT_GUIDE to the new flow.
