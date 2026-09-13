# Persistent USB ports (optional)

Generic USB-serial adapters (the controller boards used by SO-100/SO-101/Koch arms)
usually enumerate as `/dev/ttyACM0`, `/dev/ttyACM1`, etc. — the number is assigned by
enumeration order, so it can **change** across reboots or replugs, silently breaking a
saved `--robot.port`/`--teleop.port`. This is only worth doing if the user is annoyed by
re-running `lerobot-find-port` after every replug/reboot — skip it otherwise.

## 1. Get each device's stable serial number

With **both** arms connected:

```sh
for dev in /dev/ttyACM*; do
  echo "== $dev =="
  udevadm info -a -n "$dev" | grep -m1 '{serial}'
done
```

Note the serial string for the follower's device and the leader's device
(cross-reference against the ports found in Step 3 of `SKILL.md`).

## 2. Write a udev rule

Confirm with the user before writing to `/etc/udev/rules.d/` (requires `sudo`):

```sh
sudo tee /etc/udev/rules.d/99-lerobot.rules > /dev/null <<'EOF'
SUBSYSTEM=="tty", ATTRS{serial}=="<FOLLOWER_SERIAL>", SYMLINK+="lerobot_follower"
SUBSYSTEM=="tty", ATTRS{serial}=="<LEADER_SERIAL>", SYMLINK+="lerobot_leader"
EOF
```

Replace `<FOLLOWER_SERIAL>` / `<LEADER_SERIAL>` with the values from step 1.

## 3. Reload udev and reconnect

```sh
sudo udevadm control --reload-rules
sudo udevadm trigger
```

Unplug and reconnect both arms. They should now consistently appear as
`/dev/lerobot_follower` and `/dev/lerobot_leader` regardless of enumeration order —
use these paths as the permanent `--robot.port`/`--teleop.port` values instead of
`/dev/ttyACM*`.

## Two identical boards report the same serial number

Some cheap USB-serial chips (certain CH340 clones) ship with an identical, non-unique
serial burned in from the factory, which breaks the rule above (both arms would match
the same line). Work around it by matching on the physical USB port path instead:

```sh
udevadm info -a -n /dev/ttyACM0 | grep -m1 KERNELS
```

Then use `KERNELS=="<value>"` in place of `ATTRS{serial}=="..."` in the rule — this pins
the symlink to a physical USB port on the machine rather than the device's serial, so
it only works as long as that arm always plugs into that same physical port.
