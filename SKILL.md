---
name: configure-lerobot-hardware
description: >
  Configure LeRobot hardware after installation, as an interactive wizard: verify 2+
  cameras are detected with lerobot-find-cameras, find and record the USB ports for the
  leader and follower arms with lerobot-find-port, set unique motor IDs/baudrates with
  lerobot-setup-motors (for brand-new or repurposed motors), and calibrate both arms
  with lerobot-calibrate — guiding the user through the homing position and then moving
  each joint through its full range of motion. Ends with an optional live teleoperate
  smoke test. Use when a user wants to configure, set up, wire, or calibrate LeRobot
  arms/cameras, or troubleshoot camera/port/motor/calibration issues after LeRobot is
  already installed (e.g. via setup-lerobot).
compatibility: >
  Linux (native or WSL2) with LeRobot already installed and its Python environment
  active. Requires the hardware SDK extra for the arm in use (feetech for SO-100/SO-101,
  dynamixel for Koch) and a connected leader + follower arm plus at least 2 USB cameras.
metadata:
  author: your-name-here
  version: "1.0"
allowed-tools: Bash Read Write AskUserQuestion Glob Grep
---

# Configure LeRobot Hardware

This skill walks you through turning a freshly-installed LeRobot environment into a
calibrated, ready-to-record leader/follower setup, following the official hardware
guides (e.g. https://huggingface.co/docs/lerobot/so101). It's a companion to
`setup-lerobot` — run that first if `lerobot-find-cameras`, `lerobot-find-port`, etc.
aren't on the PATH yet. Deeper material lives in `reference/` — read those files with
the Read tool when instructed rather than guessing at their contents:

- `reference/troubleshooting.md` — camera, USB-permission, motor, and calibration fixes
- `reference/persistent-usb-ports.md` — optional udev rules so ports don't shift on replug/reboot

## How this wizard works

**`lerobot-find-cameras` is safe to run directly with your own Bash tool** — it's
non-interactive and finishes on its own.

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
- [ ] Step 1: Choose arm type and name each arm
- [ ] Step 2: Verify 2+ cameras are detected
- [ ] Step 3: Find the USB ports for each arm
- [ ] Step 4: (If needed) Set motor IDs and baudrate
- [ ] Step 5: Calibrate the follower arm
- [ ] Step 6: Calibrate the leader arm
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

### Step 1: Choose arm type and name each arm

Ask the user with `AskUserQuestion`:

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

Then ask for a short **id** for each arm (e.g. `my_follower_arm` / `my_leader_arm`, or
let the user pick names) — these become the `--robot.id` / `--teleop.id` values used in
every command below and name the calibration files. Store them as `$ROBOT_TYPE`,
`$FOLLOWER_ID`, `$LEADER_ID` for the rest of this walkthrough (substitute `robot`/`teleop`
type strings per the table above, e.g. `so101_follower` + `so101_leader`).

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
  which physical camera is which (e.g. "front" vs "wrist"). Note each camera's `Id`
  field (the index or `/dev/video*` path) for later use with `lerobot-record`/
  `lerobot-teleoperate`.
- **If fewer than 2 are found:** don't just report it — walk the user through
  `reference/troubleshooting.md`'s camera section (USB power, permissions, `v4l2-ctl
  --list-devices`), then re-run the command. Loop until 2+ cameras are confirmed, or the
  user explicitly says they want to proceed with fewer (record that as a caveat for the
  Step 8 summary).

### Step 3: Find the USB ports for each arm

Both arms should be connected to USB power and to the computer. Linux users may need to
grant port access first — this one you CAN run yourself (it's non-interactive), after
confirming with the user:

```sh
sudo chmod 666 /dev/ttyACM0 /dev/ttyACM1   # adjust device names as needed
```

(If `sudo` fails with "a terminal is required to authenticate" — common when your Bash
tool has no real TTY — have the user run that line themselves instead.)

Per **"How this wizard works" above, `lerobot-find-port` is interactive and must be run
by the user themselves**, once per arm. Tell them exactly this:

> Run `source <path-to-venv-or-conda-activate>` then `lerobot-find-port` in your own
> terminal. It'll list current ports, then ask you to unplug the **follower** arm's USB
> cable — do that and press Enter. It'll print the port that disappeared — tell me that
> value. Then reconnect the cable.

Record what they report as `$FOLLOWER_PORT`. Then have them repeat the same command for
the **leader** arm (unplug the leader's cable this time) and report back `$LEADER_PORT`.

If ports keep shifting between replugs/reboots and the user wants stable device paths,
point them at `reference/persistent-usb-ports.md` (optional — don't do this unless
asked, it's an extra step).

### Step 4: (If needed) Set motor IDs and baudrate

This is only required for **brand-new motors** or motors **repurposed from another
robot** — it writes a unique id + shared baudrate into each motor's EEPROM, and only
needs to happen once per motor. Ask the user with `AskUserQuestion`:

Question: "Have these arms' motors already been ID'd and calibrated before (e.g. this
is a previously-used kit), or are the motors brand new / repurposed?"
- **Already configured (Recommended if reusing a kit)** — skip straight to Step 5.
- **Brand new / repurposed motors** — run the setup below for both arms.

If setup is needed, this is interactive (per "How this wizard works") — have the user
run it themselves, for the **follower** first:

```sh
lerobot-setup-motors --robot.type=$ROBOT_TYPE --robot.port=$FOLLOWER_PORT
```

It prompts motor-by-motor ("Connect the controller board to the 'gripper' motor only
and press enter."). Tell the user to follow each prompt exactly — connect only the
named motor (not yet daisy-chained to others), press Enter, wait for the "`<motor>` id
set to N" confirmation printed in their terminal, then move to the next motor, and
report back once all motors are done (or if a prompt errors). Once all motors are done,
they reconnect the full daisy chain per the arm's assembly guide. If a step errors,
check `reference/troubleshooting.md`'s motor section before retrying.

Then repeat for the **leader**, again run by the user themselves:

```sh
lerobot-setup-motors --teleop.type=$LEADER_TYPE --teleop.port=$LEADER_PORT
```

(`$LEADER_TYPE` is the teleop counterpart, e.g. `so101_leader` if `$ROBOT_TYPE` is
`so101_follower`.)

### Step 5: Calibrate the follower arm

```sh
lerobot-calibrate --robot.type=$ROBOT_TYPE --robot.port=$FOLLOWER_PORT --robot.id=$FOLLOWER_ID
```

This is another interactive one — have the user run it themselves, and explain the
two-phase process to them first:

1. The script asks them to move the follower to its **middle/homing position** (all
   joints roughly centered in their range) and press Enter.
2. It then asks them to slowly move **every joint through its full range of motion**
   (including the gripper), so it can record min/max limits. Tell them to go slowly and
   cover the full range on each joint — this data is what makes calibration accurate.
3. The script exits on its own once done; no need to Ctrl+C.

Have them report back when it finishes (or the error text if it fails mid-way — check
`reference/troubleshooting.md`'s calibration section before suggesting a retry).

### Step 6: Calibrate the leader arm

Same procedure, run by the user themselves, for the leader:

```sh
lerobot-calibrate --teleop.type=$LEADER_TYPE --teleop.port=$LEADER_PORT --teleop.id=$LEADER_ID
```

### Step 7: Live teleoperate smoke test

Ask the user if they'd like to verify the full setup live (moving the leader should move
the follower in real time). If yes, this one streams indefinitely until `Ctrl+C` — have
the user run it themselves, not you:

```sh
lerobot-teleoperate \
    --robot.type=$ROBOT_TYPE --robot.port=$FOLLOWER_PORT --robot.id=$FOLLOWER_ID \
    --teleop.type=$LEADER_TYPE --teleop.port=$LEADER_PORT --teleop.id=$LEADER_ID
```

Tell them to move the leader arm through a few positions and confirm the follower
mirrors it smoothly (no jitter, no lag, no joints slamming into limits), then press
`Ctrl+C` to stop and report back what they observed. If motion is inverted, jittery, or
a joint maxes out immediately, that's usually a calibration issue — revisit Steps 5/6,
or check `reference/troubleshooting.md`.

This step needs only the arms — cameras aren't required yet, so skip `--robot.cameras`
here.

### Step 8: Wrap up

Summarize for the user:

- Arm type used (`$ROBOT_TYPE` / `$LEADER_TYPE`) and each arm's `id`.
- `$FOLLOWER_PORT` and `$LEADER_PORT` (and whether they set up persistent udev rules).
- How many cameras were detected and confirmed, and each one's index/path from Step 2 —
  note as a caveat if fewer than 2 were confirmed.
- Whether motor IDs were (re)configured this session or were already set.
- Calibration file locations, so the user knows where their calibration lives:
  - `~/.cache/huggingface/lerobot/calibration/robots/<robot_name>/$FOLLOWER_ID.json`
  - `~/.cache/huggingface/lerobot/calibration/teleoperators/<teleop_name>/$LEADER_ID.json`
  - (overridable via the `HF_LEROBOT_CALIBRATION` env var if the user set one)
- Whether the live teleoperate smoke test passed.
- Next step pointer: recording a dataset with `lerobot-record`, passing the
  `--robot.cameras` map built from the camera indices/paths noted in Step 2, e.g.:

  ```sh
  lerobot-record \
      --robot.type=$ROBOT_TYPE --robot.port=$FOLLOWER_PORT --robot.id=$FOLLOWER_ID \
      --robot.cameras="{ front: {type: opencv, index_or_path: 0, width: 1280, height: 720, fps: 30} }" \
      --teleop.type=$LEADER_TYPE --teleop.port=$LEADER_PORT --teleop.id=$LEADER_ID \
      --dataset.repo_id=<hf_username>/<dataset_name>
  ```

## Recalibrate only

Use this when the user just wants to redo calibration (e.g. it drifted, or a motor was
replaced) without repeating the whole wizard. It's a standalone shortcut, not part of
the numbered flow above.

Ask which arm(s) need recalibration, confirm the port hasn't changed (re-run
`lerobot-find-port` if unsure), then re-run the relevant command(s) from Step 5/6 with
the same `--robot.id`/`--teleop.id` as before — calibrating again with the same id
overwrites that arm's existing calibration file.
