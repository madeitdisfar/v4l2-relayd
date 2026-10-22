# v4l2-relayd (PipeWire camera relay)

A virtual camera for apps that read `/dev/video*` directly, such as Discord,
on the Samsung Galaxy Book4 Pro (940XGK).

## Why

The laptop's camera (OV02C10 on Intel IPU6) only gives usable video through
libcamera, which PipeWire uses. The kernel's own video devices are 48 raw IPU6
channels with no image processing. Discord's video engine skips PipeWire and
opens those kernel devices, so it gets no picture. `pw-v4l2 discord` (PipeWire's
V4L2 shim) shows the camera, but Discord's capture thread crashes on it.

## What it does

[v4l2-relayd](https://gitlab.com/vicamo/v4l2-relayd) (Canonical, made for IPU6
laptops) runs as a user service and writes into a v4l2loopback device named
**Built-in Front Camera (Virtual)**:

- While no app has the device open, it shows a black frame and the real camera
  stays off.
- When an app opens it, the relay starts reading the camera from PipeWire
  (`pipewiresrc`), converts it to YUY2 1280x720 at 30 fps, and writes it into
  the device. It stops again when the last app closes it.

The package contains:

| File | Purpose |
|---|---|
| `/usr/bin/v4l2-relayd` | The daemon, upstream 0.2.0 with `quick-restart-fixes.patch` (below) |
| `/usr/lib/systemd/user/v4l2-relayd.service` | User service: PipeWire camera → loopback (replaces upstream's system services) |
| `/usr/lib/modprobe.d/v4l2-relayd.conf` | One loopback device, `exclusive_caps=1`, label "Built-in Front Camera (Virtual)" |
| `/usr/lib/modules-load.d/v4l2-relayd.conf` | Loads `v4l2loopback` at boot |

v4l2loopback itself is part of the CachyOS kernel; on other kernels install
`v4l2loopback-dkms`.

## Patch: quick restarts

Turning the camera off and on quickly in an app left the picture black or at
a few frames per second. Two causes, both fixed in `quick-restart-fixes.patch`:

- **Reused input pipeline.** Upstream stops the camera pipeline and starts the
  same one again. After a quick restart, a reused `pipewiresrc` pipeline runs
  at 2–4 fps while another reader of the same camera gets 30. The patch frees
  the pipeline and creates a new one on every start.
- **Camera reopened while being suspended.** PipeWire's libcamera camera
  (1.6.9), reopened at the moment it is being suspended, can keep sending only
  a few frames per second. The new `-w MS` option restarts the input when it
  runs below 10 fps for that long (the service uses 1000 ms): about 3 s of
  frozen picture instead of a stuck stream. The rate is measured from the
  first frame on, since the camera's first start after login can take over a
  second; an input that sends no frame within three checks is restarted too.
  Restarts are logged in `journalctl --user -u v4l2-relayd`.

The related PipeWire stall when a camera is reopened while paused is avoided
by the camera's WirePlumber rule (`51-ipu6-camera.conf` in the
`ov02c10-26mhz-dkms` package): it suspends the camera 0.2 s after it goes idle,
without pausing it first.

## Install

```sh
makepkg -si
sudo modprobe v4l2loopback          # or reboot
systemctl --user enable --now v4l2-relayd
```

In Discord, choose **Built-in Front Camera (Virtual)**.

Settings (camera node, size, frame rate, stall timeout) are `Environment=` lines in the
service; change them with `systemctl --user edit v4l2-relayd`. The size must
be one the camera offers through PipeWire.

## Notes

- Apps that open the device just to list cameras turn the real camera on for a
  moment.
- PipeWire apps (OBS, browsers) list the virtual camera too, with " (V4L2)"
  added to its name. It works there, but goes PipeWire → relay → PipeWire;
  pick the real camera instead.
- If the camera is already in use by a PipeWire app at a different size, the
  relay can't start until that app stops.
- OBS's virtual camera also uses v4l2loopback. With one device, the two can't
  run at once; raise `devices=` in the modprobe file (and give each device a
  label) if you need both.

## Removing

```sh
systemctl --user disable --now v4l2-relayd
sudo pacman -R v4l2-relayd
sudo modprobe -r v4l2loopback       # or reboot
```

## License

v4l2-relayd is GPL-2.0-only (Canonical Ltd.), and so is
`quick-restart-fixes.patch`, as a change to it. The other packaging files in
this repository are MIT (`LICENSE`).
