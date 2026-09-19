# Persistent USB ports (serial-number binding)

Background and fallbacks for **Step 3** of `SKILL.md`. Step 3 has the happy path; come
here when something about it doesn't fit.

## Why this is not optional

The USB-serial adapters on LeRobot controller boards (WaveShare serial bus servo driver
boards for SO-100/SO-101, the U2D2 or similar for Koch) enumerate as `/dev/ttyACM0`,
`/dev/ttyACM1`, … and the **number is assigned in enumeration order**. Unplug the leader
and plug it back in, reboot, or just power the arms up in a different order, and the
numbers swap. Nothing errors — LeRobot happily opens the wrong board, so the leader's
calibration gets applied to the follower and the arm behaves nonsensically. A bimanual
rig makes this four-way and near-certain.

Every board also carries a **serial number** burned in at the factory, which never
changes. Bind the serial to a fixed `/dev` name once and every saved command keeps
working forever.

## Reading the serials

```sh
for dev in /dev/ttyACM* /dev/ttyUSB*; do
  [ -e "$dev" ] || continue
  eval "$(udevadm info -q property -n "$dev" | grep -E '^(ID_SERIAL_SHORT|ID_VENDOR_ID|ID_MODEL_ID)=')"
  printf '%-16s serial=%-20s vid:pid=%s:%s\n' "$dev" "$ID_SERIAL_SHORT" "$ID_VENDOR_ID" "$ID_MODEL_ID"
  unset ID_SERIAL_SHORT ID_VENDOR_ID ID_MODEL_ID
done
```

Typical WaveShare SO-101 board (CH343 chip):

```
/dev/ttyACM0     serial=5B3D041468         vid:pid=1a86:55d3
/dev/ttyACM1     serial=5B61035334         vid:pid=1a86:55d3
```

Other vendor ids you may see: `1a86` QinHeng (CH340/CH343), `0403` FTDI, `10c4` Silicon
Labs (CP210x), `2341`/`2e8a` Arduino/RP2040-based boards.

The raw attribute form (what a udev rule matches on) is:

```sh
udevadm info -a -n /dev/ttyACM0 | grep -m1 -E '\{serial\}|\{idVendor\}|\{idProduct\}'
```

`ATTRS{serial}` and `ENV{ID_SERIAL_SHORT}` carry the same string; the rules below use
`ATTRS{serial}`.

## Option A — `/dev/serial/by-id` (zero setup, no sudo)

The kernel already publishes a stable per-device path:

```sh
ls -l /dev/serial/by-id/
# usb-1a86_USB_Single_Serial_5B3D041468-if00 -> ../../ttyACM0
```

`/dev/serial/by-id/usb-1a86_USB_Single_Serial_5B3D041468-if00` is a perfectly good
`--robot.port` value and is already serial-bound. Use it when `sudo` isn't available or
udev isn't running. Downsides: the path doesn't say which arm it is, and it's long
enough to be error-prone in hand-typed commands — which is why Option B is the default.

## Option B — a udev rule with arm-named symlinks (recommended)

```sh
sudo tee /etc/udev/rules.d/99-lerobot.rules > /dev/null <<'EOF'
SUBSYSTEM=="tty", ATTRS{idVendor}=="1a86", ATTRS{serial}=="<FOLLOWER_SERIAL>", SYMLINK+="lerobot_follower", MODE="0666", GROUP="dialout"
SUBSYSTEM=="tty", ATTRS{idVendor}=="1a86", ATTRS{serial}=="<LEADER_SERIAL>",   SYMLINK+="lerobot_leader",   MODE="0666", GROUP="dialout"
EOF
sudo udevadm control --reload-rules && sudo udevadm trigger
sleep 2 && ls -l /dev/lerobot_*
```

Bimanual: four lines, named `lerobot_follower_left`, `lerobot_follower_right`,
`lerobot_leader_left`, `lerobot_leader_right` — matching the arm ids from Step 1d of
`SKILL.md`, so a port path and a calibration file are obviously the same arm.

Notes on the rule:

- `MODE="0666", GROUP="dialout"` sets permissions **at device creation**, which replaces
  the `sudo chmod 666 /dev/ttyACM*` dance after every replug. Adding yourself to
  `dialout` (`sudo usermod -a -G dialout $USER`, then log out/in) is the tidier variant
  if you'd rather not have world-writable device nodes.
- `SYMLINK+=` (not `NAME=`) — it adds a name alongside `ttyACM*` instead of renaming the
  device, which is what you want.
- `ATTRS{idVendor}` is belt-and-braces: it stops an unrelated device that happens to
  report the same serial string from matching. Drop it if your board's vendor id isn't
  what the rule says, or match on it loosely.
- Matching is **case-sensitive** and whitespace-sensitive. Copy serials from the command
  output, don't retype them.
- The file must be under `/etc/udev/rules.d/` and end in `.rules`. A leading number sets
  ordering; `99-` (last) is fine here.

Verify it survives a replug: unplug and reconnect every arm, then `ls -l
/dev/lerobot_*`. The symlinks should all be present, and may point at *different*
`ttyACM` numbers than before — that is the rule doing its job.

## Swapping two arms

If teleoperation shows the wrong arm responding (or left/right crossed on a bimanual
rig), you don't need to recalibrate or touch any cables: swap the two `SYMLINK+=` names
in the rules file and re-run

```sh
sudo udevadm control --reload-rules && sudo udevadm trigger
```

## Fallback: two boards report the same serial number

Some cheap CH340 clones ship with an identical, non-unique serial burned in at the
factory, so both arms match the same rule line. Check first:

```sh
for dev in /dev/ttyACM*; do udevadm info -q property -n "$dev" | sed -n 's/^ID_SERIAL_SHORT=/serial: /p'; done | sort | uniq -d
```

Any output means duplicates. Match on the **physical USB port path** instead:

```sh
udevadm info -a -n /dev/ttyACM0 | grep -m1 KERNELS      # e.g. KERNELS=="1-1.2"
```

```
SUBSYSTEM=="tty", KERNELS=="1-1.2", SYMLINK+="lerobot_follower", MODE="0666", GROUP="dialout"
SUBSYSTEM=="tty", KERNELS=="1-1.3", SYMLINK+="lerobot_leader",   MODE="0666", GROUP="dialout"
```

This pins the name to a **socket on the machine**, not to the board — so it only holds
as long as each arm is always plugged into that same physical port (and the same hub, in
the same hub port). Label the cables and the sockets. It also breaks if the hub is moved
to a different host port, since the `1-1.x` prefix changes.

## Fallback: WSL2

Serial devices reach WSL2 through `usbipd-win` (`usbipd attach --wsl --busid <id>` from
an elevated Windows PowerShell). Inside the distro they appear as `/dev/ttyACM*` as
usual, but `udev` may not be running, in which case neither `/dev/serial/by-id` nor
rules in `/etc/udev/rules.d/` materialize. Check:

```sh
ls /dev/serial/by-id/ 2>/dev/null && echo "by-id works" || echo "no by-id — udev likely not running"
systemctl is-active systemd-udevd 2>/dev/null
```

If udev isn't running, either enable systemd for the distro (`[boot]\nsystemd=true` in
`/etc/wsl.conf`, then `wsl --shutdown` from Windows) and retry, or accept raw
`/dev/ttyACM*` paths and re-run `lerobot-find-port` after each replug. Attaching the
boards in a consistent order from Windows keeps the numbering stable in practice.

## Replacing a board

The serial belongs to the board, not the arm. If a controller board is swapped out, its
symlink stops appearing — re-read the serial (top of this file) and update that one line
in `/etc/udev/rules.d/99-lerobot.rules`. Calibration files are keyed by arm `id`, not by
port, so they stay valid.
