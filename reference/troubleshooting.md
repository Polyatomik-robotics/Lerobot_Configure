# Troubleshooting

Fixes for the most common snags when configuring LeRobot hardware. Read the relevant
section when a step in `SKILL.md` fails rather than guessing.

## Cameras

**Fewer than 2 cameras detected by `lerobot-find-cameras`:**

- List raw video devices the kernel sees, independent of LeRobot:

  ```sh
  ls /dev/video*
  v4l2-ctl --list-devices   # sudo apt-get install -y v4l-utils if missing
  ```

  If a camera doesn't show up here at all, it's a hardware/OS issue, not a LeRobot one —
  try a different USB port (preferably not through an unpowered hub; webcams and USB
  cameras can draw more current than a passive hub port provides), a different cable, or
  `dmesg | tail -40` right after replugging to see if the kernel logs an enumeration
  error.

- Permission denied opening `/dev/video*`:

  ```sh
  ls -l /dev/video0                        # check group ownership, usually 'video'
  groups $USER | grep -q video || sudo usermod -a -G video $USER
  # then log out/in (or `newgrp video` in the current shell) for the group to take effect
  ```

- A single physical camera can expose **multiple** `/dev/video*` nodes (e.g. one for
  raw capture, one for metadata) — `lerobot-find-cameras` may report both. Cross-check
  the saved sample images in `outputs/captured_images/` to tell real distinct cameras
  apart from duplicate nodes of the same device.

- RealSense cameras need `pyrealsense2` — if `lerobot-find-cameras` warns "Skipping
  RealSense camera search: pyrealsense2 library not found", install it (`uv pip install
  pyrealsense2` or `pip install pyrealsense2`) if the user actually has a RealSense
  camera; otherwise ignore the warning, it's expected for OpenCV-only setups.

- **A camera is listed under `--- Detected Cameras ---` but has no file in
  `outputs/captured_images/`, or the run logs `Timed out waiting for frame`:** it's
  still genuinely detected (the listing only needs the device to open, not to deliver a
  frame) — count it toward your 2+ total — but something's preventing an actual frame
  read, almost always USB bandwidth (several cameras sharing one hub/controller). Move
  it to its own USB controller/port and re-run; if you need a working image from it
  specifically, don't just retry blindly.

- Image looks correct in `outputs/captured_images/` but the feed seems frozen/laggy
  during later live steps: usually a USB bandwidth issue (too many high-res cameras on
  one hub/controller). Try spreading cameras across different USB controllers/ports, or
  lowering resolution/fps in the `--robot.cameras` config used later with
  `lerobot-record`/`lerobot-teleoperate`.

## USB ports (arms)

**`lerobot-find-port` can't tell the ports apart / reports 0 or 2+ differences:**

- Make sure only the intended arm's cable is unplugged when prompted — a loose second
  cable, or a totally unrelated USB device (mouse dongle, USB drive) coming loose at the
  same moment, will confuse the diff.
- If the port list doesn't change at all, the OS didn't fully release the device yet —
  wait a couple of seconds after unplugging before pressing Enter, or just re-run the
  script.

**Permission denied opening `/dev/ttyACM*` / `/dev/ttyUSB*`:**

```sh
ls -l /dev/ttyACM0                          # check group, usually 'dialout'
sudo chmod 666 /dev/ttyACM0 /dev/ttyACM1    # quick fix, resets on replug/reboot
# durable fix:
groups $USER | grep -q dialout || sudo usermod -a -G dialout $USER
# then log out/in for the group to take effect
```

The udev rule from Step 3d carries `MODE="0666", GROUP="dialout"`, which fixes this
permanently — if permissions still reset on replug, the rule isn't matching that device
(see the next entry).

**Ports change every time you replug/reboot:** expected on Linux with generic USB-serial
adapters, and exactly what Step 3's serial-number binding prevents. If it's still
happening, the udev rule isn't taking effect.

**A `/dev/lerobot_*` symlink is missing after `udevadm trigger`:**

```sh
ls -l /dev/lerobot_*                                  # what actually got created
cat /etc/udev/rules.d/99-lerobot.rules                # what the rule says
for dev in /dev/ttyACM*; do udevadm info -q property -n "$dev" | sed -n 's/^ID_SERIAL_SHORT=/serial: /p'; done
udevadm test /sys/class/tty/ttyACM0 2>&1 | grep -i lerobot   # does the rule match at all?
```

- Serial mismatch is the usual cause — matching is case-sensitive, so copy the value
  from the command output rather than retyping it.
- `ATTRS{idVendor}` in the rule must match the board's real vendor id (`1a86` for the
  WaveShare CH343 boards, but `0403`/`10c4` for other adapters). Drop that clause to
  test.
- The arm must be plugged in when you run `udevadm trigger`; otherwise replug it.
- Two boards reporting the **same** serial can't be told apart by serial at all — see
  the duplicate-serial fallback in `reference/persistent-usb-ports.md`.

**The wrong arm responds / left and right are swapped (bimanual):** the symlink names
are crossed. Swap the two `SYMLINK+=` values in `/etc/udev/rules.d/99-lerobot.rules`,
re-run `sudo udevadm control --reload-rules && sudo udevadm trigger`, and re-test. No
recalibration or recabling needed — calibration is keyed by arm `id`, not by port.

## Motor setup (`lerobot-setup-motors`)

- **No response / timeout connecting to a motor:** check, in order — power supply
  actually powered and connected to the controller board, USB cable from computer to
  controller board, the 3-pin cable from the controller board to the motor, and (for
  Waveshare boards) that both jumpers are set to the `B` (USB) channel rather than the
  UART passthrough channel.
- **Make sure only one motor is connected** to the controller board at a time during
  this step, and that motor isn't daisy-chained to any other motor yet — connecting more
  than one causes bus id conflicts and the script reports the wrong motor or hangs.
- **Repurposing motors from another robot:** their old id/baudrate will conflict —
  always re-run `lerobot-setup-motors` for repurposed motors, don't assume they're ready.

## Calibration (`lerobot-calibrate`)

- **A joint won't reach the recorded min/max, or motion feels reversed after
  calibration:** almost always means a joint wasn't moved through its **full** range
  during the range-of-motion phase, or the arm wasn't at a true centered position when
  the homing step was confirmed. Re-run calibration for that arm (Step 5/6 in
  `SKILL.md`) — recalibrating with the same `--robot.id`/`--teleop.id` simply overwrites
  the old file.
- **Feetech "Incorrect status packet!" / communication errors during calibration:**
  usually a wiring or power issue that surfaces under load (multiple joints moving at
  once). Check the daisy-chain connectors and the power supply's cable, especially at
  the joint currently under the most torque (e.g. shoulder lift). The bus already
  retries transient failures automatically; repeated errors point to a physical
  connection problem, not a software one.
- **Calibration file appears empty / wrong id:** confirm `--robot.id` (or
  `--teleop.id`) was actually passed — omitting it still runs calibration but names the
  file unhelpfully. Just re-run with an explicit id; it overwrites cleanly.
- **Bimanual: calibration files named `follower_left_left.json`:** the `bi_so_follower`
  / `bi_so_leader` types append `_left` and `_right` to `--robot.id`/`--teleop.id`
  themselves, so pass the **base** name (`follower`, `leader`) — not an already-suffixed
  one. Delete the doubled-up files and re-run with the base id.
- **Bimanual: only one arm gets calibrated, or the second arm errors immediately:** the
  bimanual command drives both arms in sequence from a single invocation, so both ports
  must be live for the whole run. Check that both `--robot.left_arm_config.port` and
  `--robot.right_arm_config.port` symlinks resolve (`ls -l /dev/lerobot_*`) before
  starting.
- **Bimanual: want to recalibrate just one arm:** run it as a plain single arm against
  that arm's port with the explicit suffixed id, e.g. `--robot.type=so101_follower
  --robot.port=/dev/lerobot_follower_left --robot.id=follower_left` — it writes the same
  file the bimanual run would have.
