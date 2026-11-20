# Agent handoff — vegas26 root + LineageOS 23.2 (read this first)

This packet is for another agent continuing the work on [owner]'s phone. Do **not** act on it until you
have verified the phone's current live state. It is a device-specific record, not a generic tutorial.

## Source of truth — read in this order

1. `MotoRoot\HOWTO-vegas26-root-lineage.md` — full narrative of what we did and why.
2. `MotoRoot\ACTIONS-LOG.md` — append-only chronological log, newest entries at bottom.
3. Relevant scripts/logs in `MotoRoot\`:
   - `ab-parity-drill.ps1`, `ab-parity-rerun.ps1`, `ab-parity-2026-11-18.txt`
   - `flash-dryrun.ps1`, `gate-map.ps1`, `flash-dryrun-2026-11-18.txt`, `gate-map-2026-11-18.txt`
   - `recover-slot-a.ps1`, `recover-slot-a2.ps1`, `recover-slot-a-2026-11-18.txt`
   - `flash-lineageos.ps1`, `flash-part2.ps1`
   - `gapps-sideload.ps1`, `gapps-sideload.log`, `gapps-sideload2.log`
   - `usb-watch.ps1`, `usb-watch.log`
   - `wsl-build.sh`, `wsl-validate-build.sh`, `wsl-superimage.sh`, `wsl-superimage-nodeps.sh`
   - `flash-vegas26.sh` (tree's official flash script, also useful as reference)

## Device and user constraints

- Phone: Motorola Moto G Power 2026, `XT2617V`, codename `vegas26`, serial `[serial redacted]`.
- [owner] is at the phone and will tap UI prompts when asked. The agent runs PC/adb/fastboot commands.
- Windows `pwsh`. **Inline PowerShell commands are rejected by this harness** — write a `.ps1` file
  and run `pwsh -NoProfile -ExecutionPolicy Bypass -File ...`.
- No admin rights in the normal shell. Zadig was run elevated once with UAC approval to bind WinUSB
  to the ADB interface (`VID_18D1&PID_D001`). The fastbootd interface (`PID_4EE0`) previously showed
  `CM_PROB_FAILED_INSTALL`; replugging the cable often re-enumerates it.
- **No BROM rescue path.** A damaged bootloader chain can permanently brick the phone.
- [owner]'s instruction: "try not to break my phone"; do lots of validation before writes; keep logging.

## Last verified state (as of 2026-11-18; re-verify before acting)

- Slot A: LineageOS 23.2 self-built, `userdebug/test-keys`, `ro.lineage.version =
  23.2-20261118-UNOFFICIAL-vegas26`, patch `2026-11-01`, kernel `5.15.189-...`, unlocked,
  verifiedboot state `orange`. Wi-Fi worked (connected to [wifi-ssid], 390 Mbps).
- Slot B: **untouched rollback lever** — stock `-4-3` with Magisk 31.1 root, modules
  `wifi_ota_guard`, `wifi_carrier_debloat`. Revert with `fastboot --set-active=b`.
- Boot chain on slot A was verified byte-exact against `MotoRoot\lineage-release\SHA256SUMS`.
- Gapps: [owner] reported "gapps installed" after manually finishing the recovery install, but this was
  **not re-verified with adb** after the last disconnect. Treat as unknown until checked.
- Root on LineageOS / slot A: **not done yet**. Magisk is only on slot B.
- At the last check (2026-11-20), `adb devices` showed no phone connected. Start by getting USB/adb
  back; do not assume any state from this document is still live.

## First actions for a continuing agent

1. Re-establish adb/fastboot and identify current mode/slot:
   ```sh
   adb devices -l
   adb shell getprop ro.lineage.version
   adb shell getprop ro.boot.slot_suffix
   adb shell getprop sys.boot_completed
   fastboot devices
   fastboot getvar current-slot
   fastboot getvar is-userspace
   fastboot getvar securestate
   ```
2. If booted into LineageOS, check Gapps/root:
   ```sh
   adb shell 'ls /system/etc/g.prop >/dev/null 2>&1 && echo GAPPS_PRESENT || echo GAPPS_ABSENT'
   adb shell 'pm list packages | grep -E "com.google.android.gms|com.android.vending"'
   adb shell 'id; su -c id'     # root not expected on slot A yet
   ```
3. If recovery is needed: [owner] must tap **Advanced → Enable ADB** on every recovery boot;
   sideload additionally needs **Apply update → Apply from ADB**.
4. Append every action/outcome to `MotoRoot\ACTIONS-LOG.md`.

## Critical mistakes already made — do not repeat

- **Never flash the pre-slot-A-copy backup images.** They are `-4-2`, not `-4-3`. Flashing
  `vbmeta_system_a`, `dtbo_a`, `vendor_boot_a` from that backup set caused rollback index 5 and made
  slot A unbootable. Recovery used live slot-B `-4-3` copies:
  `live-vbmeta_system_a.img`, `dtbo_b.img`, `vendor_boot_b.img`.
- A "Preflash validation failed" on `boot_a` was also caused by a stale `-4-2` backup, not by a
  Motorola gate quirk. The gate correctly validates image↔vbmeta descriptors.
- Flash `vbmeta`/`vbmeta_system` first. Keep rollback indexes at 7.
- Only flash the approved images: `boot`, `init_boot`, `vendor_boot`, `dtbo`, `vbmeta`,
  `vbmeta_system`, and the 5 logical partitions in `super`. No firmware partitions.
- Upstream: all 8 repos were unchanged on `lineage-23.2`, HEAD 2026-11-16, no releases/tags. Re-check
  before suggesting an update.

## Remaining likely task: root LineageOS on slot A

Slot A is a `userdebug` build, so `adb root` works. Because the bootloader gate will reject a
Magisk-patched `init_boot_a` (hash mismatch with vbmeta descriptors), the intended method is:

1. Install `MotoRoot\Magisk-v31.1.apk`.
2. In Magisk: **Install → Select and Patch a File** → choose the Lineage `init_boot.img`
   (copy from `MotoRoot\lineage-release\init_boot.img` to the phone).
3. Push the patched image back to `/data/local/tmp/init_boot_patched.img`.
4. From root adb, `dd` it to `/dev/block/by-name/init_boot_a` (set RW, `conv=fsync`, flush, verify
   hash), then reboot and verify `su -c 'magisk -v'`.
5. Do **not** lose slot B: it remains the rollback lever until the new root is proven.

## Tone / autonomy expectations

- Ask [owner] before any write/flash/wipe. Validate first, explain the plan, then act.
- Keep logs and be explicit about uncertainty. If Gapps/root status is unknown, verify it rather than
  assuming it from this handoff.
- If USB/fastbootd breaks, recreate the known-good path: replug cable; if needed, elevated Zadig/WinUSB
  binding with [owner]'s help; use recovery adb where possible. Do not improvise firmware flashing.

## Credits / upstream sources

This work stands on other people's research, code and threads. Credit and thanks to the following.
None of their binaries are bundled in the public handoff repo.

**Original guide and community thread**

- [parsimo2010/moto-g-power-2026-straight-talk-wifi-root-guide](https://github.com/parsimo2010/moto-g-power-2026-straight-talk-wifi-root-guide) — the original Wi-Fi-only setup / temp-root / bootloader-unlock / Magisk guide and guide kit (by u/parsimo).
- [Reddit r/StraightTalk announcement thread](https://www.reddit.com/r/StraightTalk/comments/1xwgm37/i_rooted_a_walmart_straight_talk_moto_g_power/) by u/parsimo. The `-4-3` and Magisk wrinkle was worked out in the comments by u/angel9990 and u/mclcs, with keygen/OTA input from u/CORRUPT27, u/jghost1337 and u/newofficemusic. The optional modem-level carrier unlock was reported there by u/No_Car_6879 and confirmed on `-4-3` by u/mclcs.

**Bootloader unlock, temporary root, Magisk**

- [Maikyxd/val-protocol](https://github.com/Maikyxd/val-protocol) — LK patch and unlock-key generation. Its own upstream credits: [kasnria001/pwnage24mtk](https://github.com/kasnria001/pwnage24mtk), [R0rt1z2/liblk](https://github.com/R0rt1z2/liblk), [R0rt1z2/lkpatcher](https://github.com/R0rt1z2/lkpatcher).
- [diabl0w/DFRoot](https://github.com/diabl0w/DFRoot) v4.7 — temporary root (CVE-2026-43284).
- [tiann/KernelSU](https://github.com/tiann/KernelSU) 3.3.1 — temporary root.
- [topjohnwu/Magisk](https://github.com/topjohnwu/Magisk) 31.1 — persistent root and the systemless modules.
- [googlesamples/android-testdpc](https://github.com/googlesamples/android-testdpc) — Test DPC provisioning QR used at setup.

**LineageOS 23.2**

- [jimmielin/android_device_motorola_vegas26](https://github.com/jimmielin/android_device_motorola_vegas26) (`lineage-23.2`) plus the companion `jimmielin` trees `android_device_motorola_vegas26-kernels`, `android_device_motorola_mt6835-common`, `android_vendor_motorola_vegas26`, `android_vendor_motorola_mt6835-common-vegas26` — the device/vendor trees this ROM was built from.
- [LineageOS](https://github.com/LineageOS) — `android_hardware_mediatek`, `android_device_mediatek_sepolicy_vndr`, `android_hardware_motorola`, and the AOSP/LineageOS platform.
- [MindTheGapps/16.0.0-arm64](https://github.com/MindTheGapps/16.0.0-arm64) — the Gapps package.
- [XDA thread — UNOFFICIAL Lineage OS/Lineage Recovery G Play 2026](https://xdaforums.com/t/unofficial-lineage-os-lineage-recovery-g-play-2026.4801191/) by u/Significant-Noise530 — sibling XT2615V device, reference only (not flashed).
- [Reddit r/androidroot — SUCCESS: unlocked Moto G Play 2026 (XT2615V) bootloader](https://www.reddit.com/r/androidroot/comments/1vkb0zl/success_unlocked_moto_g_play_2026_xt2615v/) by u/Significant-Noise530 — Val Protocol notes on a sibling MT6835 device.

**Carrier / SIM unlock**

- [andrewwm/vegas-freedom](https://github.com/andrewwm/vegas-freedom) (commit `ed78c7e`) — the bootloader + modem (`md1img`) SIM-unlock patcher used here. The modem half is adapted from [Articfox291/kansas-modem-unlock](https://github.com/Articfox291/kansas-modem-unlock); the bootloader half from [crabcakes97/Valhalla](https://github.com/crabcakes97/Valhalla) (built on val-protocol + liblk).

**Other background (reference only — not used on this phone)**

- [jesussaves-dev/moto-g-power-2026-root](https://github.com/jesussaves-dev/moto-g-power-2026-root) — GSI / custom-AVB-key route.
- [fuckyoumoto.xyz](https://fuckyoumoto.xyz) — background on other Motorola MTK unlock mechanics.
- Google (Android Platform Tools) and Motorola (signed Windows USB driver).
