# StatusBack

Bring back the status-bar icons that Samsung One UI 7/8 hides on the home screen — VoLTE / Wi-Fi
calling, Bluetooth, alarm, Do Not Disturb, hotspot, and more — each with its own toggle in a small
settings app. A libxposed module (API 102), scoped to `com.android.systemui` only.

- Changes apply **instantly**: no reboot, no SystemUI restart, no re-flash
- Nothing is patched on disk; disable the module and everything reverts
- Your own `icon_blacklist` (System UI Tuner / `Settings.Secure`) still wins

## Requirements

- Rooted device with Vector 2.2+ (or another API-102 libxposed framework) — Vector needs Zygisk
  (e.g. NeoZygisk with KernelSU)
- Samsung One UI with the `…statusbar.phone.ui.StatusBarIconControllerImpl` layout — developed on a
  Galaxy S22 Ultra (One UI 8 / Android 16); other builds may rename classes, in which case the hook
  logs an error and does nothing

## Install

1. Install this APK.
2. In Vector Manager, enable **StatusBack** with scope **System UI** (`com.android.systemui`).
3. Grant the one-time permission (root shell or adb):
   ```sh
   adb shell pm grant io.github.fragtion.statusback android.permission.WRITE_SECURE_SETTINGS
   ```
4. Reboot once so SystemUI loads the module.
5. Open **StatusBack** and flip the toggles.

## How it works

One UI's `StatusBarIconControllerImpl.hideBySimplification(IconManager, String slot)` returns
`true` for the home and keyguard bars unless the slot is in
`config_status_bar_system_icon_allowlist`. StatusBack hooks it and returns `false` for the slots
you enabled — but only when the calling `IconManager` reports `mLocation == HOME`, so the keyguard
and every other call path stay untouched.

The enabled slots are stored as a comma-separated list in `Settings.Secure`
(`statusback_restore_slots`). SystemUI watches that key with a `ContentObserver` and calls
`refreshIconGroups()` on change, which is why toggles take effect live.

Defaults before you save anything: `ims_volte`, `ims_volte2`, `bluetooth`, `bluetooth_connected`.

## Slots

| Slot | Shows |
|---|---|
| `ims_volte`, `ims_volte2` | VoLTE / Wi-Fi calling badge (SIM 1 / 2) |
| `bluetooth`, `bluetooth_connected` | Bluetooth on / device connected |
| `alarm_clock` | Alarm set |
| `mute`, `volume` | Silent / vibrate, sound-mode indicator |
| `zen` | Do Not Disturb / Modes |
| `rotate` | Rotation lock |
| `hotspot`, `bt_tethering` | Hotspot, Bluetooth tethering |
| `data_saver` | Data Saver |
| `location` | Location in use |
| `cast`, `screen_record`, `connected_display` | Cast / Smart View, screen recording, external display / DeX |
| `satellite` | Satellite connectivity |
| `camera`, `microphone`, `sensors_off` | Privacy indicators / sensors-off |

`wifi`, `mobile` and `mobile2` are already allowed by One UI and are not listed.

## Limitations

- A toggle only un-hides a slot SystemUI actually receives. If Samsung's IMS service doesn't publish
  `ims_volte` (or there's no second SIM for `ims_volte2`), there is nothing to show.
- Some carriers render Wi-Fi calling as a variant of the Wi-Fi *signal* icon
  (`USE_WIFI_CALLING_ICON`); that path isn't touched by this hook.
- Slot names come from SystemUI dumps on one device; `volume` in particular is unconfirmed.

## Troubleshooting

List the slots your SystemUI currently has and their visibility:

```sh
adb shell
su
D=com.android.systemui/.SystemUIService
dumpsys activity service $D StatusBarIconControllerImpl > /data/local/tmp/s.txt
sed -n "s/.*slot='\([^']*)'.*visibleState=\([A-Z]*\).*/\1 \2/p" /data/local/tmp/s.txt
```

The module logs each distinct HOME slot it sees (`HOME slot: <name> -> shown|hidden`) through the
framework log: check Vector Manager → Logs, or `adb logcat | grep StatusBack`.

If SystemUI misbehaves, disable the module in Vector and reboot.

## License

MIT — source and license at <https://github.com/fragtion/StatusBack>.
