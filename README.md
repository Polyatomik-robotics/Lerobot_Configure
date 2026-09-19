# configure-lerobot-hardware

A [Claude Code Skill](https://docs.claude.com/en/docs/claude-code/skills) that walks an
agent through configuring [Hugging Face LeRobot](https://github.com/huggingface/lerobot)
hardware after installation: picking a single-arm or bimanual layout and naming each arm,
verifying 2+ cameras are detected with `lerobot-find-cameras`, finding each arm's USB
port with `lerobot-find-port` and binding it to a stable `/dev/lerobot_*` name by the
board's serial number, setting motor IDs/baudrate for new or repurposed motors with
`lerobot-setup-motors`, and calibrating every arm with `lerobot-calibrate` — guiding the
user through the homing position and then a full range-of-motion sweep for each joint.
Finishes with an optional live teleoperate smoke test.

It's a companion to a LeRobot install skill (e.g.
[`setup-lerobot`](https://github.com/Polyatomik-robotics/Lerobot_Setup)) — run that
first if the `lerobot-*` CLIs aren't on the PATH yet.

Modeled on the same pattern as `setup-lerobot`: a single `SKILL.md` runbook the agent
executes step by step, with deeper reference material split into separate files it
reads on demand.

## Install

Once this repo is pushed to GitHub, install it into any project with the
[`skills` CLI](https://github.com/vercel-labs/skills):

```sh
npx skills add <owner>/<repo>
```

This drops `SKILL.md` (and `reference/`) into `.claude/skills/configure-lerobot-hardware/`
in your project (or `~/.claude/skills/` with `-g` for a global install). Claude Code
picks it up automatically next session — then just ask your agent to configure your
LeRobot cameras and arms, and it will follow the skill.

## Local development / testing

While iterating on this skill before publishing it, symlink it straight into your global
skills directory instead of copy-pasting:

```sh
ln -s "$(pwd)" ~/.claude/skills/configure-lerobot-hardware
```

## Structure

```
SKILL.md                          # the runbook: checklist + step-by-step wizard
reference/
  troubleshooting.md              # camera/USB-permission/motor/calibration/symlink fixes
  persistent-usb-ports.md         # serial-number port binding: udev rules, /dev/serial/by-id,
                                  # duplicate-serial and WSL2 fallbacks
```

## Notes

- Targets native Linux and WSL2, matching the LeRobot hardware docs.
- Covers SO-100, SO-101, and Koch leader/follower kits by default (all share the same
  `lerobot-find-port` / `lerobot-setup-motors` / `lerobot-calibrate` CLI surface); other
  custom robot configs work too if the user supplies their own `--robot.type`/
  `--teleop.type` strings.
- Handles both single-arm (`leader` / `follower`) and bimanual (`leader_left`,
  `leader_right`, `follower_left`, `follower_right`) setups. Bimanual SO-100/SO-101 rigs
  use LeRobot's `bi_so_follower` / `bi_so_leader` types, which drive both arms of a side
  from one command; the skill falls back to four independent single-arm configs when
  those types aren't available.
- Port binding is part of the main flow, not an optional extra: each controller board's
  serial number is read with `udevadm` and pinned to an arm-named symlink
  (`/dev/lerobot_follower_left`, …) via a udev rule that also sets permissions — so
  replugging or rebooting can't shuffle which arm is which.
- Assumes LeRobot and the relevant hardware extra (`feetech` or `dynamixel`) are already
  installed — this skill checks for that in Step 0 and points to an install skill if not.
- Update `metadata.author` in `SKILL.md` before publishing.
