# Moto G Power 2026 (XT2617V / `vegas26`) — unlock, root, back up, and install self-built LineageOS 23.2

This is the full sequence we actually followed on **serial [serial redacted]**, written so someone else can
repeat it. It is heavily device- and firmware-specific. **Do not adapt it blindly to another model.**

---

## 0. Ground rules and hardware facts

| Item | Value |
|---|---|
| Device | Motorola Moto G Power 2026, `XT2617V`, codename `vegas26` |
| Serial | `[serial redacted]` |
| SoC | MediaTek MT6835 (Dimensity 6300), GKI kernel 5.15 |
| Stock firmware used | `W1WEOS36.48-22-3-3-4-3` (must match on the phone) |
| Bootloader string | `MBM-3.1-vegas_og_vext-5d518d-W1WEOS36.48-22-3-3-4-3-7806ed` |
| Stock security patch | boot/vendor `2026-06-01` |
| Slots | A/B (Virtual A/B + compression), `init_boot`, `super` = 11,475,615,744 bytes |
| Critical warning | **No known BROM rescue path.** A damaged bootloader chain is a permanent brick. |
| Rollback indexes | `vbmeta` = 7, `vbmeta_system` = 7 on both slots (after our prep) |

Firmware rules we obeyed:

- Only ever flash `boot`, `init_boot`, `vendor_boot`, `dtbo`, `vbmeta`, `vbmeta_system` and the logical
  partitions inside `super`. **Never** flash `preloader`, `lk`, `tee`, `gz`, `md1img`, etc.
  (The one exception was the LK unlock patch, done deliberately on `lk_a` from the guide kit.)
- Never flash firmware from a different stock build. The vendor blobs in the build came from
  `-4-3`, so the phone's bootloader-chain firmware must also be `-4-3`.
- Flash `vbmeta` and `vbmeta_system` **first** — the Motorola bootloader validates every other image
  against the descriptors in them.
- The tree's official flash script ships with the release (`flash-vegas26.sh`), and there is no stock
  firmware package online for this build — so **back up every partition before touching anything**.

---

## 1. Verify the guide kit and the phone

Before any writes:

1. Verify the downloaded guide/kit against its `SHA256SUMS`.
2. Confirm on the phone (stock Android, adb):
   ```sh
   adb shell getprop ro.boot.slot_suffix          # _a or _b
   adb shell getprop ro.build.fingerprint          # must be W1WEOS36.48-22-3-3-4-3
   adb shell getprop ro.boot.bootloader            # must end -4-3
   adb reboot bootloader
   fastboot getvar is-userspace                    # no (bootloader fastboot)
   fastboot getvar sku                             # XT2617V
   fastboot getvar current-slot
   fastboot getvar securestate
   fastboot getvar version-bootloader              # -4-3
   ```

---

## 2. Bootloader unlock (Val Protocol / LK patch)

The official unlock path is not available for this SKU; the guide uses a **patched LK** on `lk_a`.

- Guide values used: secret `[unlock secret redacted]`, key `[unlock key redacted]`.
- Patched image: `lk.unlock-serial.img` (SHA-256 starts `914acfce…`).
- Written to `lk_a` from the bootloader/fastboot path per the guide.
- Verified afterwards: `fastboot getvar securestate` → `flashing_unlocked`.

This is device-specific; do not reuse the patched LK on another handset.

---

## 3. Backups (do this before root or any flashing)

All backups were SHA-256 verified and kept private:

| Backup | Contents | Size |
|---|---|---|
| `backups-2026-11-17\fullbackup\` | 94 partition images + `sha256.csv` | 12.54 GB |
| `backups-2026-11-17\data-backup\db\` | 52 data tars | 14.22 GB |
| `private-backups\` | stock `lk_a`, `init_boot_b`, `misc`, `seccfg` | — |

Method (high level): boot stock rooted (or recovery), read every `/dev/block/by-name/<partition>`
with `dd`, pull with `adb`, then verify each file against a generated `sha256.csv`. Data was archived
with `tar`; only `/data/system` and `/data/vendor` produced harmless socket-file warnings.

---

## 4. Root stock Android on slot B with Magisk 31.1

Slot B was chosen as the permanent rollback lever.

1. Install the Magisk app, let it patch **the stock `init_boot`** for slot B.
2. Write the patched image to `init_boot_b` (`dd` from a root shell; `fastboot` is blocked by the
   Motorola preflash gate for boot-chain images whose hash no longer matches `vbmeta`).
   Patched `init_boot_b` SHA-256 starts `747304c0…`.
3. Fix the Magisk environment via the app dialog, then verify across reboots:
   ```sh
   adb shell su -c id                 # uid=0, context=u:r:magisk:s0
   adb shell su -c 'magisk -v'        # 31.1:MAGISK:R
   ```
   `su` lives at `/product/bin/su`.
4. Installed modules: `wifi_ota_guard`, `wifi_carrier_debloat`.
5. Disabled `com.android.devicelockcontroller`; removed DFRoot/KernelSU apps.
6. Set `settings put global ota_disable_automatic_update 1`; kept Wi-Fi on.

---

## 5. Make slot A a safe, stock twin (the rollback drill)

Stock ships with only slot A fully populated, so we normalised both slots *before* installing LineageOS.

1. Copied firmware from live slot B → slot A using the tree script
   `mt6835-common/tools/copy-firmware-to-other-slot.sh` (dry-run first, then real):
   `lk tee gz sspm spmfw mcupm dpm scp vcp pi_img md1img connsys_bt logo mcf_ota`.
   `tzapp` was deliberately **not** copied. All readbacks verified.
2. Copied OS images from slot B → slot A: `vbmeta vbmeta_system boot vendor_boot dtbo`, plus the
   stock `init_boot` from `private-backups`.
3. **Rollback-index trap:** the first `--set-active=a` boot failed and fell back to B. Cause: slot A's
   `vbmeta` had rollback index **5** while the device was at **7**. Re-copying slot B's `vbmeta`
   images (index 7) fixed it. Slot A then booted stock `-4-3` cleanly.
4. A/B parity + rollback drill (as root on slot B, hashing `/dev/block/by-name/*_a` vs `*_b`):
   - all listed firmware partitions **SAME**; `tzapp` DIFF (expected);
   - `vbmeta vbmeta_system boot vendor_boot dtbo` **SAME**; `init_boot` DIFF (Magisk only on B);
   - both slots' rollback indexes = 7;
   - `--set-active=b` → slot B booted with root/modules/Wi-Fi; `--set-active=a` → slot A booted stock.
     **The revert path was proven before any ROM was flashed.**

---

## 6. Build LineageOS 23.2 from source

Environment: WSL2 Ubuntu 22.04, 24 GB RAM + 16 GB swap (first attempt OOM-killed at a 20 GB cap),
24 threads, ~118 GB source tree, ~723 GB free disk.

1. Add the 8 repos via `.repo/local_manifests/vegas26.xml` (all on branch `lineage-23.2`):
   - `jimmielin/android_device_motorola_vegas26`
   - `jimmielin/android_device_motorola_vegas26-kernels`
   - `jimmielin/android_device_motorola_mt6835-common`
   - `jimmielin/android_vendor_motorola_vegas26`
   - `jimmielin/android_vendor_motorola_mt6835-common-vegas26`
   - `LineageOS/android_hardware_mediatek`
   - `LineageOS/android_device_mediatek_sepolicy_vndr`
   - `LineageOS/android_hardware_motorola`
2. Sync and build:
   ```sh
   repo sync -c -j...
   source build/envsetup.sh
   lunch lineage_vegas26-bp4a-userdebug
   m -j12 bacon
   ```
3. The PC rebooted at 97 % — **no work was lost**; ninja state survived and `m -j12 bacon` resumed
   incrementally. Final: exit 0, 11:53 wall time, 171,103 edges, peak RSS ~15 GB, `out/` ~97 GB.

### Build validation (all passed)

- `vbmeta` rollback index **7**, flags 1; `vbmeta_system` rollback **7**.
- Security patches: boot/vendor `2026-06-01`; system/product/system_ext `2026-11-01`.
- `boot.img` kernel is byte-identical to stock `Image.gz`; `vendor_boot` DTB identical to stock
  `mt6835.dtb`; `dtbo` payload identical (only the AVB footer/fingerprint prop is re-signed).
- `avbtool verify_image` passes for boot/init_boot/vendor_boot/dtbo.
- Fingerprint in `build.prop`: `motorola/lineage_vegas26/vegas26:16/BP4A.251205.006/eng.ferg:userdebug/test-keys`
  (test-keys; the tree author has only tested release keys — watch for TEE/trustlet oddities).

Staged 12 images (2.71 GB) in `lineage-release\` with a `SHA256SUMS` file:
`vbmeta vbmeta_system boot init_boot vendor_boot dtbo super_empty system system_ext product vendor vendor_dlkm`.

---

## 7. Flash LineageOS to slot A

Preconditions rechecked in bootloader: `is-userspace=no`, `sku=XT2617V`, `current-slot=a`,
`securestate=flashing_unlocked`, bootloader `-4-3`, `avb_custom_key: not found`.

### Part 1 — boot chain (bootloader fastboot)

```sh
fastboot flash vbmeta_a        vbmeta.img
fastboot flash vbmeta_system_a vbmeta_system.img
fastboot flash boot_a          boot.img
fastboot flash init_boot_a     init_boot.img
fastboot flash vendor_boot_a   vendor_boot.img
fastboot flash dtbo_a          dtbo.img
```

All six succeeded. Verified afterwards from root adb in recovery: every image matched the release
hash exactly (`vbmeta` partitions are 8 MB, only the first 64 KB is written — expected).

### Part 2 — logical partitions (fastbootd)

```sh
fastboot reboot fastboot                 # is-userspace: yes
fastboot wipe-super super_empty.img
fastboot flash system       system.img
fastboot flash system_ext   system_ext.img
fastboot flash product      product.img
fastboot flash vendor       vendor.img
fastboot flash vendor_dlkm  vendor_dlkm.img
fastboot -w                              # erase stock encrypted data
```

### Part 3 — Google apps (before first boot!)

```sh
fastboot reboot recovery
# On the phone: Advanced > Enable ADB, then Apply update > Apply from ADB
adb sideload MindTheGapps-16.0.0-arm64-20261023_034858.zip
# Accept the "unsigned with this build's keys" warning
fastboot reboot      # or "Reboot system now" in recovery
```

MindTheGapps `16.0.0-arm64-20261023_034858.zip` SHA-256:
`6929593937cd85541228e4e5db9cbb5455338bf33c75eabf610de632ac155810`.
**If you skip Gapps at this point, you must format data again to add them later.**

### Verified result on the phone

```
ro.lineage.version      = 23.2-20261118-UNOFFICIAL-vegas26
ro.build.display.id     = BP4A.251205.006 test-keys
ro.build.type           = userdebug
ro.boot.slot_suffix     = _a
ro.build.version.security_patch = 2026-11-01
ro.boot.flash.locked    = 0   (unlocked)
ro.boot.verifiedbootstate = orange
kernel                  = 5.15.189-android13-8-...
Wi-Fi                   = connected, 11ac, 390 Mbps
```

Note: `ro.build.fingerprint` still shows the Motorola stock string — the device tree deliberately
overrides it. Use `ro.lineage.version` to confirm the ROM.

---

## 8. Problems we hit, and the fixes

These are the "gotchas" worth passing on:

1. **Stale backup images bricked slot A temporarily.** A dry-run flashed the *pre-slot-A-copy*
   backup images (`-4-2`, `vbmeta_system` rollback 5) instead of the live slot B copies
   (`-4-3`, rollback 7). Slot A would not boot. Fix: reflash the correct `-4-3`
   `vbmeta_system_a`/`dtbo_a`/`vendor_boot_a` (a watchdog script did this automatically once USB
   returned). **Lesson: never flash backup slot-A images after you have copied slot B's firmware over.**
2. **`fastboot flash boot_a` failed with "Preflash validation failed"** even with an "identical"
   image — it was actually the stale `-4-2` backup, whose hash no longer matched the `-4-3` vbmeta
   descriptors. The gate behaves correctly; feed it matching images.
3. **Safe mode + airplane mode** appeared after the bootloops; a normal reboot cleared
   `mSafeMode=true`, airplane mode and Wi-Fi recovered.
4. **Windows fastbootd driver**: `VID_18D1&PID_4EE0` came up as `CM_PROB_FAILED_INSTALL`, so
   `fastboot` could not see fastbootd. Zadig was used (elevated) to bind WinUSB to the ADB interface
   (`PID_D001`). Replugging the USB-C cable also repeatedly re-enumerated the device.
5. **Recovery ADB** must be enabled on every boot: Lineage Recovery → *Advanced → Enable ADB*.
   Sideload additionally requires *Apply update → Apply from ADB*.
6. **Gapps sideload was slow/unstable** (one run stalled at 46 %, another returned `0.00x` with a
   "fuse unable to be written / timed out" message). [owner] eventually completed/installed it from
   the phone; as of the last report Gapps are installed, though we have not re-verified with adb
   since the phone was disconnected.
7. **`super.img` fallback:** a full `m bacon superimage` rebuild was started as insurance (it
   rebuilds ~124k edges, ~2-3 h); it was not needed because fastbootd succeeded.

---

## 9. Logging and upstream tracking

Running record: `MotoRoot\ACTIONS-LOG.md` (append-only, all of the above in detail).

Other logs/artifacts in `MotoRoot\`:

- `ab-parity-2026-11-18.txt` — A/B parity and rollback drill
- `gate-map-2026-11-18.txt`, `flash-dryrun-2026-11-18.txt` — preflash-gate dry runs
- `recover-slot-a-2026-11-18.txt` — incident recovery watchdog output
- `flash-lineage-2026-11-18.txt`, `gapps-sideload*.log` — flash/sideload output
- `wsl-validate-build.log` — build validation
- `lineage-release\SHA256SUMS` — hashes of the 12 staged images

Upstream was re-checked twice: all 8 repos remained at the same `lineage-23.2` HEAD
(newest commit 2026-11-16); no releases and no tags were published.

---

## 10. Remaining work after this point

- **Re-root the LineageOS install** (slot B still has Magisk; slot A currently does not):
  install `Magisk-v31.1.apk`, patch **the Lineage `init_boot.img`**, then write it to
  `init_boot_a`. Because the preflash gate rejects boot-chain images whose hash differs from the
  vbmeta descriptors, use root adb (`adb root` works on this userdebug build) and `dd` the patched
  image to `/dev/block/by-name/init_boot_a`, then verify `su -c 'magisk -v'` after reboot.
- Re-verify that Gapps (Play Store / GMS) are present once the phone is back on adb.
- Keep slot B untouched as the instant revert: `fastboot --set-active=b`.
- Re-check the GitHub repos/releases periodically; update by flashing new images (the ROM's own
  updater and VoLTE/VoWiFi are not yet verified in this tree).

---

## 11. Credits / upstream sources

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
