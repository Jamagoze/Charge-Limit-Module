# Charge Limit (Magisk / KernelSU / APatch module)

Stops charging at a battery percentage you choose (1-99%, default 80) and resumes a few percent lower. Pure shell and sysfs: no Zygisk, no APK, no accessibility service.

Current version: **v1.8**

## Does it only work on HyperOS?

No. Nothing in it is HyperOS-specific. It needs:

- Root with Magisk 20.4 or newer, KernelSU or APatch
- A kernel that exposes a writable switch that turns charging off

The module tries these switches in order and keeps the first one that really stops charging:

| Node | Stop | Resume |
|---|---|---|
| `power_supply/battery/charging_enabled` | 0 | 1 |
| `power_supply/battery/battery_charging_enabled` | 0 | 1 |
| `qcom-battery/battery_charging_enabled` | 0 | 1 |
| `power_supply/battery/charge_disable` | 1 | 0 |
| `power_supply/battery/input_suspend` | 1 | 0 |
| `qcom-battery/input_suspend` | 1 | 0 |
| `power_supply/usb/input_suspend` | 1 | 0 |
| `power_supply/main/input_suspend` | 1 | 0 |
| `power_supply/battery/batt_slate_mode` | 1 | 0 |

Likely to work: Xiaomi / Redmi / POCO (MIUI, HyperOS) and other Qualcomm devices with these nodes, including custom ROMs.
May need a node added: Pixel, OnePlus / Oppo, Motorola, MediaTek.
May not work: devices whose kernel has no writable charging switch, and some Samsung One UI builds.

Confirmed device: Redmi Pad SE, HyperOS, Android 14 (Snapdragon 680, charger chip SC89890H, switch `qcom-battery/input_suspend`).

To check another device, run `su -c stopcharge nodes` and look for a file with "charge" or "suspend" in its name.

## Install

1. Install `stop_charge_limit.zip` from the Magisk, KernelSU or APatch app (Modules, Install from storage).
2. Reboot.
3. Set your limit: `su -c stopcharge 80`

Updating: flash the new zip over the old one and reboot. Your settings are kept.

## KernelSU / APatch

Works the same as on Magisk. Newer KernelSU does not mount a module's `system/` folder unless a metamodule is installed, so the module also places a `stopcharge` wrapper in the manager's own bin folder (`/data/adb/ksu/bin` or `/data/adb/ap/bin`) at every boot. If `su -c stopcharge` says "not found", run it by full path: `su -c /data/adb/ksu/bin/stopcharge` (APatch: `/data/adb/ap/bin/stopcharge`), or `su -c sh /data/adb/modules/stopcharge/system/bin/stopcharge`.

Scripts run under the manager's BusyBox, so process listing uses the system `ps` first and falls back to BusyBox.

## Commands (Termux, root)

| Command | What it does |
|---|---|
| `su -c stopcharge 70` | Set the limit (1-99). Applies immediately. |
| `su -c stopcharge off` | Turn the limit off (normal charging). |
| `su -c stopcharge` | Show limit, battery, mode, paused state, node. |
| `su -c stopcharge gap 3` | Resume charging this many percent below the limit (1-50, default 2). |
| `su -c stopcharge wake rtc` | Deep-sleep friendly mode using the RTC alarm (default). |
| `su -c stopcharge wake on` | Keep the CPU awake while plugged in (most reliable, uses a little more power). |
| `su -c stopcharge wake off` | No wake handling. Only acts while the tablet is awake. |
| `su -c stopcharge hw on` / `hw off` | Use a kernel threshold node if the device has one (see below). |
| `su -c stopcharge nodes` | List battery files, for diagnosing a new device. |
| `su -c stopcharge rtctest` | Test whether the RTC alarm wakes the device from sleep. |

Settings live in `/data/adb/stopcharge/config`:

```
LIMIT=80
GAP=2
WAKE=2
HW=0
```

`WAKE`: 2 = RTC alarm, 1 = wakelock, 0 = off. After editing the file by hand, run `su -c sh /data/adb/modules/stopcharge/check.sh` to apply it now.

## How it works

- A small timer runs the check every 5-10 s while charging and every 60 s otherwise. It only counts while the tablet is awake, so it adds no wake-ups in deep sleep.
- Deep sleep: while the CPU is suspended no software can run, so the module always keeps an RTC alarm armed. While charging it is set for about half of the estimated time to the limit (re-planned on every wake). While paused it wakes about once an hour to check for the resume level. While unplugged it wakes every 30 min when far below the limit, every 10 min when within 12%, and every 5 min within 3%.
- If the battery is already above the limit, charging is blocked in advance, even when unplugged, so plugging in does not charge. `stopcharge off` disables this.
- If the RTC is unusable, it falls back to the wakelock method on its own.
- Hardware mode: if the kernel has a real percentage threshold node (`charge_control_end_threshold`, `batt_full_capacity`), `hw on` writes the limit there and the module does nothing else.
- Charging control: stop and resume go through the node found in the table above. The detected node is stored in `/data/adb/stopcharge/node`.
- Android's battery event log (`logcat` battery_level) was tried in v1.2-v1.6 but never fires on HyperOS, so it is no longer used.

## Files

| Path | Purpose |
|---|---|
| `/data/adb/stopcharge/config` | Your settings |
| `/data/adb/stopcharge/node` | Detected charging switch |
| `/data/adb/stopcharge/state` | 1 if charging is currently paused |
| `/data/adb/stopcharge/daemon.pid`, `rtc.pid` | Running processes |
| `/data/adb/stopcharge/rtcfail` | Exists if the RTC alarm was unusable |
| `/data/adb/stopcharge/wl` | Exists while the wakelock is held |

`log.txt` is no longer written since v1.4 and can be deleted.

## Notes

- HyperOS "Charging protection" or similar battery settings can fight the module. Turn them off.
- `charge_control_limit` on the Redmi Pad SE stops the charger chip but does not resume until the cable is replugged, so the module does not use it.
- On the Redmi Pad SE, `voltage_max` and `constant_charge_current_max` are read-only, so there is no voltage-based limit.

## Uninstall

Remove the module in your root manager and reboot. It restores charging, releases the wakelock, clears the RTC alarm and deletes `/data/adb/stopcharge`.

## Version history

**v1.8**
- KernelSU and APatch support: manager-agnostic installer stub, `stopcharge` wrapper placed in the manager's bin folder, BusyBox-safe process listing, `Root` line in the status.
- module.prop author set to Jamagoze.

**v1.7**
- Replaced the battery-event listener (never fired on HyperOS) with an awake-only timer plus an always-armed RTC alarm for deep sleep.
- Charging is blocked in advance when the battery is already above the limit.
- `stopcharge off` to disable the limit.

**v1.6**
- `stopcharge` status now shows diagnostics: whether the event listener and RTC waiter are running, when the last battery event and RTC wake happened, when the last check ran, and the planned alarm. Single overwritten files, no log growth.
- RTC alarm now wakes at half of the estimated time to the limit (was 80%), capped at 30 minutes, so it lands earlier.
- Safer lock handling so a killed check can never block later checks.

**v1.5**
- RTC alarm mode for deep sleep, now the default. The tablet is woken shortly before the limit and while paused.
- `stopcharge rtctest` to check the RTC on a device.
- Falls back to the wakelock method if the RTC is unusable.

**v1.4**
- All logging removed (no `log.txt`, no `stopcharge log`).
- `stopcharge wake on|off` and hardware mode (`hw on|off`).
- `stopcharge nodes` for diagnostics.

**v1.3**
- Wakelock held while plugged in or paused, so the limit works with the screen off.

**v1.2**
- Event-driven: replaces polling with a `logcat` listener on battery events. No CPU use while unplugged.
- `stopcharge <limit>` applies immediately.

**v1.1**
- Lighter polling: shell builtins instead of spawning `cat`, adaptive intervals (5 s near the limit, 60 s unplugged).
- Log trimmed to the latest lines instead of being wiped.

**v1.0**
- First release. Automatic charging-switch detection, limit 1-99%, resume gap, `stopcharge` command, polling every 10-20 s.
