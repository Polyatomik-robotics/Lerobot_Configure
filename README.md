# configure-lerobot-hardware

A [Claude Code Skill](https://docs.claude.com/en/docs/claude-code/skills) that walks an
agent through configuring [Hugging Face LeRobot](https://github.com/huggingface/lerobot)
hardware after installation: verifying 2+ cameras are detected with
`lerobot-find-cameras`, finding the USB ports for the leader and follower arms with
`lerobot-find-port`, setting motor IDs/baudrate for new or repurposed motors with
`lerobot-setup-motors`, and calibrating both arms with `lerobot-calibrate` — guiding the
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
  troubleshooting.md              # camera/USB-permission/motor/calibration fixes
  persistent-usb-ports.md         # optional udev rules so ports don't shift on replug/reboot
```

## Notes

- Targets native Linux and WSL2, matching the LeRobot hardware docs.
- Covers SO-100, SO-101, and Koch leader/follower kits by default (all share the same
  `lerobot-find-port` / `lerobot-setup-motors` / `lerobot-calibrate` CLI surface); other
  custom robot configs work too if the user supplies their own `--robot.type`/
  `--teleop.type` strings.
- Assumes LeRobot and the relevant hardware extra (`feetech` or `dynamixel`) are already
  installed — this skill checks for that in Step 0 and points to an install skill if not.
- Update `metadata.author` in `SKILL.md` before publishing.
