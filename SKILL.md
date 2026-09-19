---
name: configure-lerobot-hardware
description: >
  Configure LeRobot hardware after installation, as an interactive wizard: choose a
  single-arm or bimanual setup and name each arm, verify cameras are detected with
  lerobot-find-cameras, find each arm's USB port with lerobot-find-port and bind it to a
  stable /dev name by the board's serial number (so ports never shuffle on replug or
  reboot), set unique motor IDs/baudrates with lerobot-setup-motors (for brand-new or
  repurposed motors), and calibrate every arm with lerobot-calibrate — guiding the user
  through the homing position and then moving each joint through its full range of
  motion. Ends with an optional live teleoperate smoke test. Use when a user wants to
  configure, set up, wire, or calibrate LeRobot arms/cameras, or troubleshoot
  camera/port/motor/calibration issues after LeRobot is already installed (e.g. via
  setup-lerobot).
compatibility: >
  Linux (native or WSL2) with LeRobot already installed and its Python environment
  active. Requires the hardware SDK extra for the arm in use (feetech for SO-100/SO-101,
  dynamixel for Koch) and a connected leader + follower arm (or two of each, for a
  bimanual setup) plus at least 2 USB cameras.
metadata:
  author: your-name-here
  version: "1.1"
allowed-tools: Bash Read Write AskUserQuestion Glob Grep
---

# Configure LeRobot Hardware

This skill walks you through turning a freshly-installed LeRobot environment into a
calibrated, ready-to-record leader/follower setup — single-arm or bimanual — following
the official hardware guides (e.g. https://huggingface.co/docs/lerobot/so101). It's a
companion to `setup-lerobot` — run that first if `lerobot-find-cameras`,
`lerobot-find-port`, etc. aren't on the PATH yet. Deeper material lives in `reference/` —
read those files with the Read tool when instructed rather than guessing at their
contents:

- `reference/troubleshooting.md` — camera, USB-permission, motor, and calibration fixes
- `reference/persistent-usb-ports.md` — full detail on serial-number port binding
  (udev rules, `/dev/serial/by-id`, duplicate-serial and WSL2 fallbacks)

## How this wizard works

**`lerobot-find-cameras` is safe to run directly with your own Bash tool** — it's
non-interactive and finishes on its own. So are all the `udevadm` / `ls` port-inspection
commands in Step 3.

**`lerobot-find-port`, `lerobot-setup-motors`, `lerobot-calibrate`, and
`lerobot-teleoperate` are NOT** — each one calls Python's `input()` (or streams
indefinitely) expecting a human to perform a physical action — unplug a cable, connect
one motor, move a joint, press Ctrl+C — in real time, between prompts. Running one of
these yourself through your Bash tool (or having the user pipe it through a `!`
passthrough) will not work: there is no live TTY attached to pause on, so the process
either hits `EOFError: EOF when reading a line` immediately or hangs until your tool
timeout kills it. Confirmed by hand: both paths fail with that exact `EOFError`.

Instead, for each of these four commands: tell the user the exact command to run and
what to expect from the prompts, and have **them** run it in a terminal window they
control directly (a normal terminal app, not mediated through this chat) — then have
them report back the result (the port found, "motor set" confirmations, calibration
success, what they observed live) so you can record it and move to the next step.

## Instructions

Copy this checklist and track your progress as you go:

```
Implementation Progress:
- [ ] Step 0: Preflight checks
- [ ] Step 1: Single-arm or bimanual, arm type, and arm names
- [ ] Step 2: Verify 2+ cameras are detected
- [ ] Step 3: Find each arm's USB port and bind it to a stable name by serial number
- [ ] Step 4: (If needed) Set motor IDs and baudrate
- [ ] Step 5: Calibrate the follower side
- [ ] Step 6: Calibrate the leader side
- [ ] Step 7: Live teleoperate smoke test
- [ ] Step 8: Wrap up
```

### Step 0: Preflight checks

```sh
uname -s                                             # must be Linux
command -v lerobot-find-cameras >/dev/null && echo "find-cameras: found" || echo "find-cameras: MISSING"
command -v lerobot-find-port    >/dev/null && echo "find-port: found"    || echo "find-port: MISSING"
command -v lerobot-setup-motors >/dev/null && echo "setup-motors: found" || echo "setup-motors: MISSING"
command -v lerobot-calibrate    >/dev/null && echo "calibrate: found"    || echo "calibrate: MISSING"
command -v lerobot-teleoperate  >/dev/null && echo "teleoperate: found"  || echo "teleoperate: MISSING"
python -c "import cv2" 2>/dev/null && echo "opencv: found" || echo "opencv: MISSING"
```

If any CLI is missing, tell the user their LeRobot environment isn't active or isn't
fully installed — point them at the `setup-lerobot` skill, or `source .venv/bin/activate`
/ `conda activate lerobot` if they just forgot to activate it. Don't proceed until all
five CLIs resolve.

### Step 1: Single-arm or bimanual, arm type, and arm names

**1a. Ask how many arms**, with `AskUserQuestion`:

Question: "Is this a single-arm setup or a bimanual (two-arm) setup?"
- **Single arm (Recommended)** — one leader + one follower (2 boards, 2 USB ports).
  Arms are named `leader` and `follower`.
- **Bimanual** — two leaders + two followers (4 boards, 4 USB ports). Arms are named
  `leader_left` / `leader_right` and `follower_left` / `follower_right`.

This answer drives the rest of the wizard: how many ports you find in Step 3, how many
arms you set up and calibrate in Steps 4–6, and the command shape in Step 7.

**1b. Ask which arm kit**, with `AskUserQuestion`:

Question: "Which arm kit are you configuring?"
- **SO-101 (Recommended)** — current flagship kit (`so101_follower` / `so101_leader`).
  Needs the `feetech` extra.
- **SO-100** — predecessor kit, same CLI surface (`so100_follower` / `so100_leader`).
  Needs the `feetech` extra.
- **Koch** — (`koch_follower` / `koch_leader`). Needs the `dynamixel` extra.
- **Other / custom** — ask the user directly for the exact `--robot.type` /
  `--teleop.type` strings they use (a third-party or self-registered robot config).

Check the matching extra is installed and offer to install it if not (confirm before
running):

```sh
python -c "import scservo_sdk" 2>/dev/null || echo "feetech extra missing"   # SO-100/SO-101
python -c "import dynamixel_sdk" 2>/dev/null || echo "dynamixel extra missing"  # Koch
# to install (pick the matching extra, from inside the lerobot checkout):
uv pip install -e ".[feetech]"     # or ".[dynamixel]"
# or: pip install -e ".[feetech]"  # or ".[dynamixel]"
```

**1c. Resolve the config types.** For a **bimanual SO-100/SO-101** setup, LeRobot has
first-class bimanual types that drive both arms of a side from one command:

| Setup | `--robot.type` (followers) | `--teleop.type` (leaders) |
|---|---|---|
| Single, SO-101 | `so101_follower` | `so101_leader` |
| Single, SO-100 | `so100_follower` | `so100_leader` |
| Single, Koch | `koch_follower` | `koch_leader` |
| Bimanual, SO-100/SO-101 | `bi_so_follower` | `bi_so_leader` |

Confirm the bimanual types exist in the installed version before relying on them:

```sh
python -c "import lerobot.robots.bi_so_follower, lerobot.teleoperators.bi_so_leader; print('bimanual types: available')" 2>/dev/null \
  || echo "bimanual types: NOT available — fall back to per-arm single configs"
```

If they're **not** available (older LeRobot, or a Koch/custom bimanual rig), fall back to
treating each of the four arms as its own single-arm device: run every command in Steps
4–6 once per arm with its own `--robot.port`/`--robot.id` (or `--teleop.*`), and tell the
user that `lerobot-teleoperate` in Step 7 can only drive one leader/follower pair at a
time in that mode.

**1d. Fix the arm names (ids).** Defaults, which you should offer and let the user
override:

| Setup | Leader id(s) | Follower id(s) |
|---|---|---|
| Single | `leader` | `follower` |
| Bimanual | `leader_left`, `leader_right` | `follower_left`, `follower_right` |

Ids are used verbatim in shell commands and calibration filenames, so keep them
lowercase with underscores — `follower_left` rather than `Follower-Left`, even if the
user thinks of it by the prettier name.

**Important for the bimanual `bi_so_*` types:** you pass **one** id per side, and
LeRobot appends `_left` / `_right` itself. So `--robot.id=follower` produces the two
calibration files `follower_left.json` and `follower_right.json` — pass the base name
(`follower` / `leader`), *not* `follower_left`, or you'll end up with
`follower_left_left.json`.

Store for the rest of this walkthrough:

- `$ROBOT_TYPE` / `$LEADER_TYPE` — from the table in 1c.
- `$FOLLOWER_ID` / `$LEADER_ID` — the ids (base names, for bimanual).
- Single: `$FOLLOWER_PORT`, `$LEADER_PORT`.
- Bimanual: `$FOLLOWER_LEFT_PORT`, `$FOLLOWER_RIGHT_PORT`, `$LEADER_LEFT_PORT`,
  `$LEADER_RIGHT_PORT`.

### Step 2: Verify 2+ cameras are detected

Ask the user to plug in all cameras now if they haven't, then run:

```sh
lerobot-find-cameras
```

Count the detected cameras (each one prints a `Camera #N:` block under
`--- Detected Cameras ---`):

```sh
lerobot-find-cameras 2>/dev/null | grep -c "^Camera #"
```

- **If 2 or more are found:** the command also saves a sample image per camera to
  `outputs/captured_images/` — tell the user where, so they can open them and confirm
  which physical camera is which (e.g. "front" vs "wrist", or "left wrist" vs "right
  wrist" for a bimanual rig). Note each camera's `Id` field (the index or `/dev/video*`
  path) for later use with `lerobot-record`/`lerobot-teleoperate`.
- **If fewer than 2 are found:** don't just report it — walk the user through
  `reference/troubleshooting.md`'s camera section (USB power, permissions, `v4l2-ctl
  --list-devices`), then re-run the command. Loop until 2+ cameras are confirmed, or the
  user explicitly says they want to proceed with fewer (record that as a caveat for the
  Step 8 summary).

A bimanual rig usually wants more cameras than a single-arm one (a wrist camera per
follower plus a shared front view) — mention that, but don't block on it.

### Step 3: Find each arm's USB port and bind it to a stable name by serial number

`/dev/ttyACM0` / `/dev/ttyACM1` numbering is assigned in enumeration order, so it
reshuffles whenever a cable is replugged or the machine reboots — which silently swaps
leader and follower and makes every saved command wrong. **Don't stop at the raw
`/dev/ttyACM*` path.** Bind each board's unique serial number to a stable `/dev` name,
then use that name everywhere from here on.

**3a. Grant port access.** All arms should be connected to USB power and to the
computer. This one you CAN run yourself (it's non-interactive), after confirming with
the user:

```sh
sudo chmod 666 /dev/ttyACM*   # temporary; the udev rule in 3d makes this permanent
```

(If `sudo` fails with "a terminal is required to authenticate" — common when your Bash
tool has no real TTY — have the user run that line themselves instead.)

**3b. Identify which port is which arm.** Per **"How this wizard works" above,
`lerobot-find-port` is interactive and must be run by the user themselves**, once per
arm. Tell them exactly this (naming the specific arm each time — for bimanual, do this
four times, e.g. "the **left follower**"):

> Run `source <path-to-venv-or-conda-activate>` then `lerobot-find-port` in your own
> terminal. It'll list current ports, then ask you to unplug the **<arm name>** arm's USB
> cable — do that and press Enter. It'll print the port that disappeared — tell me that
> value. Then reconnect the cable.

Record each reported port against its arm name. Leave every arm plugged in when done.

**3c. Map each port to its board's serial number.** Run this yourself — it's
non-interactive:

```sh
for dev in /dev/ttyACM* /dev/ttyUSB*; do
  [ -e "$dev" ] || continue
  eval "$(udevadm info -q property -n "$dev" | grep -E '^(ID_SERIAL_SHORT|ID_VENDOR_ID|ID_MODEL_ID)=')"
  printf '%-16s serial=%-20s vid:pid=%s:%s\n' "$dev" "$ID_SERIAL_SHORT" "$ID_VENDOR_ID" "$ID_MODEL_ID"
  unset ID_SERIAL_SHORT ID_VENDOR_ID ID_MODEL_ID
done
ls -l /dev/serial/by-id/ 2>/dev/null   # same info, as ready-made stable paths
```

WaveShare serial bus servo driver boards (the SO-100/SO-101 controller) use a CH343
chip, `vid:pid=1a86:55d3`, and each board carries a distinct serial like `5B3D041468`.
Cross-reference the serial against the arm name from 3b, and **read the serials back to
the user** so they can confirm the mapping before you write any rule.

If two boards report the **same** serial (some CH340 clones), stop and read
`reference/persistent-usb-ports.md` — there's a physical-USB-port fallback there.

**3d. Write the udev rule.** This is the payoff: friendly, arm-named device paths that
survive replugs and reboots, plus permanent permissions (no more `chmod` after every
replug). Confirm with the user before writing to `/etc/udev/rules.d/` (requires `sudo`),
and use their arm names from Step 1d:

```sh
# Single-arm — substitute each <SERIAL> from 3c
sudo tee /etc/udev/rules.d/99-lerobot.rules > /dev/null <<'EOF'
SUBSYSTEM=="tty", ATTRS{idVendor}=="1a86", ATTRS{serial}=="<FOLLOWER_SERIAL>", SYMLINK+="lerobot_follower", MODE="0666", GROUP="dialout"
SUBSYSTEM=="tty", ATTRS{idVendor}=="1a86", ATTRS{serial}=="<LEADER_SERIAL>",   SYMLINK+="lerobot_leader",   MODE="0666", GROUP="dialout"
EOF
```

```sh
# Bimanual — one line per arm, named to match the ids from Step 1d
sudo tee /etc/udev/rules.d/99-lerobot.rules > /dev/null <<'EOF'
SUBSYSTEM=="tty", ATTRS{idVendor}=="1a86", ATTRS{serial}=="<FOLLOWER_LEFT_SERIAL>",  SYMLINK+="lerobot_follower_left",  MODE="0666", GROUP="dialout"
SUBSYSTEM=="tty", ATTRS{idVendor}=="1a86", ATTRS{serial}=="<FOLLOWER_RIGHT_SERIAL>", SYMLINK+="lerobot_follower_right", MODE="0666", GROUP="dialout"
SUBSYSTEM=="tty", ATTRS{idVendor}=="1a86", ATTRS{serial}=="<LEADER_LEFT_SERIAL>",    SYMLINK+="lerobot_leader_left",    MODE="0666", GROUP="dialout"
SUBSYSTEM=="tty", ATTRS{idVendor}=="1a86", ATTRS{serial}=="<LEADER_RIGHT_SERIAL>",   SYMLINK+="lerobot_leader_right",   MODE="0666", GROUP="dialout"
EOF
```

(Adjust `idVendor` if 3c reported something other than `1a86` — e.g. `0403` for FTDI,
`10c4` for Silicon Labs. Dropping the `ATTRS{idVendor}` clause entirely also works; it's
there to guard against an unrelated device sharing a serial string.)

**3e. Reload udev and verify.** Run this yourself:

```sh
sudo udevadm control --reload-rules && sudo udevadm trigger
sleep 2
ls -l /dev/lerobot_*
```

Every arm must show up with its own symlink pointing at a `ttyACM*` device. If a symlink
is missing, re-check the serial in the rule against 3c's output (the match is
case-sensitive), then have the user replug that arm and re-run the `ls`.

**3f. Use the stable names from here on.** Set the port variables to the symlinks, not
the raw `ttyACM` paths:

```sh
# Single
FOLLOWER_PORT=/dev/lerobot_follower
LEADER_PORT=/dev/lerobot_leader

# Bimanual
FOLLOWER_LEFT_PORT=/dev/lerobot_follower_left
FOLLOWER_RIGHT_PORT=/dev/lerobot_follower_right
LEADER_LEFT_PORT=/dev/lerobot_leader_left
LEADER_RIGHT_PORT=/dev/lerobot_leader_right
```

**Then have the user sanity-check the binding:** unplug and replug every arm (or reboot),
re-run `ls -l /dev/lerobot_*`, and confirm each symlink still resolves — possibly to a
*different* `ttyACM` number, which is exactly the point. Every command in Steps 4–8 uses
the symlink, so it keeps working regardless.

If udev rules aren't an option (WSL2 without udev, no `sudo`), fall back to the
`/dev/serial/by-id/usb-...-if00` paths printed in 3c — they're equally stable, just
uglier. `reference/persistent-usb-ports.md` covers both fallbacks in detail.

### Step 4: (If needed) Set motor IDs and baudrate

This is only required for **brand-new motors** or motors **repurposed from another
robot** — it writes a unique id + shared baudrate into each motor's EEPROM, and only
needs to happen once per motor. Ask the user with `AskUserQuestion`:

Question: "Have these arms' motors already been ID'd and calibrated before (e.g. this
is a previously-used kit), or are the motors brand new / repurposed?"
- **Already configured (Recommended if reusing a kit)** — skip straight to Step 5.
- **Brand new / repurposed motors** — run the setup below for every arm.

If setup is needed, this is interactive (per "How this wizard works") — have the user
run it themselves, **followers first**:

```sh
# Single
lerobot-setup-motors --robot.type=$ROBOT_TYPE --robot.port=$FOLLOWER_PORT

# Bimanual (bi_so_follower walks the left arm's motors, then the right arm's)
lerobot-setup-motors --robot.type=$ROBOT_TYPE \
    --robot.left_arm_config.port=$FOLLOWER_LEFT_PORT \
    --robot.right_arm_config.port=$FOLLOWER_RIGHT_PORT
```

It prompts motor-by-motor ("Connect the controller board to the 'gripper' motor only
and press enter."). Tell the user to follow each prompt exactly — connect only the
named motor (not yet daisy-chained to others), press Enter, wait for the "`<motor>` id
set to N" confirmation printed in their terminal, then move to the next motor, and
report back once all motors are done (or if a prompt errors). In the bimanual case warn
them the run covers **both** arms back to back: the left arm's six motors, then the
right arm's — they'll need to keep track of which arm they're working on. Once all
motors are done, they reconnect the full daisy chain per the arm's assembly guide. If a
step errors, check `reference/troubleshooting.md`'s motor section before retrying.

Then repeat for the **leaders**, again run by the user themselves:

```sh
# Single
lerobot-setup-motors --teleop.type=$LEADER_TYPE --teleop.port=$LEADER_PORT

# Bimanual
lerobot-setup-motors --teleop.type=$LEADER_TYPE \
    --teleop.left_arm_config.port=$LEADER_LEFT_PORT \
    --teleop.right_arm_config.port=$LEADER_RIGHT_PORT
```

### Step 5: Calibrate the follower side

```sh
# Single
lerobot-calibrate --robot.type=$ROBOT_TYPE --robot.port=$FOLLOWER_PORT --robot.id=$FOLLOWER_ID

# Bimanual (one command, both arms in sequence; ids become ${FOLLOWER_ID}_left / _right)
lerobot-calibrate --robot.type=$ROBOT_TYPE \
    --robot.left_arm_config.port=$FOLLOWER_LEFT_PORT \
    --robot.right_arm_config.port=$FOLLOWER_RIGHT_PORT \
    --robot.id=$FOLLOWER_ID
```

This is another interactive one — have the user run it themselves, and explain the
two-phase process to them first:

1. The script asks them to move the follower to its **middle/homing position** (all
   joints roughly centered in their range) and press Enter.
2. It then asks them to slowly move **every joint through its full range of motion**
   (including the gripper), so it can record min/max limits. Tell them to go slowly and
   cover the full range on each joint — this data is what makes calibration accurate.
3. The script exits on its own once done; no need to Ctrl+C.

For bimanual, that whole two-phase sequence runs **twice** — left arm first, then right.
Tell the user to watch the prompts so they calibrate the arm the script is actually
talking to.

Have them report back when it finishes (or the error text if it fails mid-way — check
`reference/troubleshooting.md`'s calibration section before suggesting a retry).

### Step 6: Calibrate the leader side

Same procedure, run by the user themselves, for the leader(s):

```sh
# Single
lerobot-calibrate --teleop.type=$LEADER_TYPE --teleop.port=$LEADER_PORT --teleop.id=$LEADER_ID

# Bimanual
lerobot-calibrate --teleop.type=$LEADER_TYPE \
    --teleop.left_arm_config.port=$LEADER_LEFT_PORT \
    --teleop.right_arm_config.port=$LEADER_RIGHT_PORT \
    --teleop.id=$LEADER_ID
```

### Step 7: Live teleoperate smoke test

Ask the user if they'd like to verify the full setup live (moving a leader should move
the matching follower in real time). If yes, this one streams indefinitely until
`Ctrl+C` — have the user run it themselves, not you:

```sh
# Single
lerobot-teleoperate \
    --robot.type=$ROBOT_TYPE --robot.port=$FOLLOWER_PORT --robot.id=$FOLLOWER_ID \
    --teleop.type=$LEADER_TYPE --teleop.port=$LEADER_PORT --teleop.id=$LEADER_ID
```

```sh
# Bimanual — both pairs driven at once
lerobot-teleoperate \
    --robot.type=$ROBOT_TYPE \
    --robot.left_arm_config.port=$FOLLOWER_LEFT_PORT \
    --robot.right_arm_config.port=$FOLLOWER_RIGHT_PORT \
    --robot.id=$FOLLOWER_ID \
    --teleop.type=$LEADER_TYPE \
    --teleop.left_arm_config.port=$LEADER_LEFT_PORT \
    --teleop.right_arm_config.port=$LEADER_RIGHT_PORT \
    --teleop.id=$LEADER_ID
```

Tell them to move each leader arm through a few positions and confirm the matching
follower mirrors it smoothly (no jitter, no lag, no joints slamming into limits), then
press `Ctrl+C` to stop and report back what they observed. If motion is inverted,
jittery, or a joint maxes out immediately, that's usually a calibration issue — revisit
Steps 5/6, or check `reference/troubleshooting.md`.

**Bimanual sides swapped** (the left leader drives the right follower): the left/right
port assignments are crossed. Because the ports are now serial-bound symlinks, fix it
once in `/etc/udev/rules.d/99-lerobot.rules` — swap the two `SYMLINK+=` names on the
affected pair's lines, re-run Step 3e — and it stays fixed.

This step needs only the arms — cameras aren't required yet, so skip `--robot.cameras`
here.

### Step 8: Wrap up

Summarize for the user:

- Single-arm or bimanual, arm type used (`$ROBOT_TYPE` / `$LEADER_TYPE`), and each arm's
  `id`.
- The stable port mapping — each arm's name, its `/dev/lerobot_*` symlink, and the
  board serial behind it — plus the fact that `/etc/udev/rules.d/99-lerobot.rules` is
  what keeps it stable across replugs/reboots (and where to edit it if a board is ever
  replaced).
- How many cameras were detected and confirmed, and each one's index/path from Step 2 —
  note as a caveat if fewer than 2 were confirmed.
- Whether motor IDs were (re)configured this session or were already set.
- Calibration file locations, so the user knows where their calibration lives:
  - `~/.cache/huggingface/lerobot/calibration/robots/<robot_name>/$FOLLOWER_ID.json`
  - `~/.cache/huggingface/lerobot/calibration/teleoperators/<teleop_name>/$LEADER_ID.json`
  - Bimanual writes two files per side, suffixed `_left` / `_right`, e.g.
    `.../robots/so_follower/${FOLLOWER_ID}_left.json` — note that `<robot_name>` is the
    underlying arm class (`so_follower`), not the `bi_so_follower` type string.
  - (overridable via the `HF_LEROBOT_CALIBRATION` env var if the user set one)
- Whether the live teleoperate smoke test passed.
- Next step pointer: recording a dataset with `lerobot-record`, passing the
  `--robot.cameras` map built from the camera indices/paths noted in Step 2, e.g.:

  ```sh
  # Single
  lerobot-record \
      --robot.type=$ROBOT_TYPE --robot.port=$FOLLOWER_PORT --robot.id=$FOLLOWER_ID \
      --robot.cameras="{ front: {type: opencv, index_or_path: 0, width: 1280, height: 720, fps: 30} }" \
      --teleop.type=$LEADER_TYPE --teleop.port=$LEADER_PORT --teleop.id=$LEADER_ID \
      --dataset.repo_id=<hf_username>/<dataset_name>
  ```

  ```sh
  # Bimanual — per-arm cameras are prefixed left_/right_ in the dataset;
  # top-level --robot.cameras keys stay unprefixed (use that for a shared front view)
  lerobot-record \
      --robot.type=$ROBOT_TYPE \
      --robot.left_arm_config.port=$FOLLOWER_LEFT_PORT \
      --robot.right_arm_config.port=$FOLLOWER_RIGHT_PORT \
      --robot.id=$FOLLOWER_ID \
      --robot.left_arm_config.cameras="{ wrist: {type: opencv, index_or_path: 1, width: 640, height: 480, fps: 30} }" \
      --robot.right_arm_config.cameras="{ wrist: {type: opencv, index_or_path: 2, width: 640, height: 480, fps: 30} }" \
      --robot.cameras="{ front: {type: opencv, index_or_path: 0, width: 1280, height: 720, fps: 30} }" \
      --teleop.type=$LEADER_TYPE \
      --teleop.left_arm_config.port=$LEADER_LEFT_PORT \
      --teleop.right_arm_config.port=$LEADER_RIGHT_PORT \
      --teleop.id=$LEADER_ID \
      --dataset.repo_id=<hf_username>/<dataset_name>
  ```

## Recalibrate only

Use this when the user just wants to redo calibration (e.g. it drifted, or a motor was
replaced) without repeating the whole wizard. It's a standalone shortcut, not part of
the numbered flow above.

Ask which arm(s) need recalibration, confirm the ports still resolve (`ls -l
/dev/lerobot_*` — if a symlink is missing, the board was replaced or the rule is stale,
so redo Step 3), then re-run the relevant command(s) from Step 5/6 with the same
`--robot.id`/`--teleop.id` as before — calibrating again with the same id overwrites
that arm's existing calibration file.

For a bimanual rig, the `bi_so_*` command recalibrates **both** arms of that side. To
redo just one arm, run it as a single-arm device against that arm's port, with the
explicit suffixed id — e.g. `--robot.type=so101_follower
--robot.port=/dev/lerobot_follower_left --robot.id=${FOLLOWER_ID}_left` — which writes
exactly the same calibration file the bimanual run would have.
