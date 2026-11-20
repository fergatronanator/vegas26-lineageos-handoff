# Action log — vegas26 (Moto G Power 2026, XT2617V, serial [serial redacted])

Append-only record of what was done to the phone/PC. Newest entries at the bottom.

## 2026-11-17 — stock root path (completed earlier)
- Guide kit verified (SHA256SUMS), Val Protocol LK patch (secret `[unlock secret redacted]`), bootloader unlocked
  (`securestate=flashing_unlocked`), patched LK live on `lk_a`.
- Magisk 31.1: patched `init_boot_b` written by `dd` (hash 747304c0…), env fixed via app dialog,
  persistent root verified across reboots (`su` at `/product/bin/su`).
- Modules: `wifi_ota_guard`, `wifi_carrier_debloat` active; `com.android.devicelockcontroller`
  disabled; DFRoot/KernelSU apps removed; Wi-Fi on; `ota_disable_automatic_update=1`.
- Full partition backup: 94 images / 12.54 GB, SHA-256 verified. Data backup: 52 tars / 14.22 GB,
  verified (socket-file warnings only on `/data/system`, `/data/vendor`).

## 2026-11-18 — slot A prep (overnight)
- Ran tree `copy-firmware-to-other-slot.sh` dry-run, then real run: slot B firmware → slot A
  (`lk tee gz sspm spmfw mcupm dpm scp vcp pi_img md1img connsys_bt logo mcf_ota`), all readback OK.
  `tzapp` deliberately not copied.
- First `--set-active=a` boot failed; bootloader fell back to slot B (~200 s).
  Root cause: slot A `vbmeta` rollback index **5**, device at **7** → rollback protection reject.
- Copied slot B OS images to slot A (`vbmeta vbmeta_system boot vendor_boot dtbo`) + stock
  `init_boot` from `private-backups`. Readback verified. Slot A vbmeta now rollback index 7.
- Slot A boots stock `-4-3` unlocked, Wi-Fi on, OTA flag intact.
- LineageOS 23.2 source synced (118 GB), local manifest `vegas26.xml`, lunch target
  `lineage_vegas26-bp4a-userdebug`.
- Build attempt 1: soong_build OOM-killed at 20 GB WSL cap (peak 19.7 GB). `.wslconfig` → 24 GB
  RAM / 16 GB swap; rebuild `m -j12 bacon` (log `~/build.log`).
- MindTheGapps 16.0.0-arm64-20261023_034858.zip downloaded + SHA-256 verified.
- Flash checks confirmed in bootloader: `is-userspace=no`, `sku=XT2617V`, `current-slot=a`,
  `securestate=flashing_unlocked`, bootloader `…-4-3-…`; `avb_custom_key: not found`.

## 2026-11-18 — A/B parity + rollback drill + gate mapping
- Rollback drill PASSED: `--set-active=b` → slot B booted stock `-4-3` with Magisk root
  (`31.1:MAGISK:R`), modules `wifi_carrier_debloat`/`wifi_ota_guard`, Wi-Fi on; then returned to slot A.
- Parity (hashes read as root on slot B): all firmware `lk tee gz sspm spmfw mcupm dpm scp vcp pi_img
  md1img connsys_bt connsys_gnss connsys_wifi logo mcf_ota` **SAME** A vs B; `tzapp` DIFF (expected).
  All OS `vbmeta vbmeta_system boot vendor_boot dtbo` **SAME**; `init_boot` DIFF (Magisk on B, stock on A).
- Live rollback index: slot A/B `vbmeta` = 7, `vbmeta_system` = 7 (both).
- Upstream check: all 8 tree repos at same HEAD as GitHub `lineage-23.2`; no new commits,
  no releases, no tags since 2026-11-16.
- Flash-path dry run: `fastboot flash vbmeta_a` (identical OEM) OK; `boot_a` → **Preflash validation
  failed**. Correction: the failure was my mistake, not a gate quirk — I flashed the stale
  `-4-2` backup `boot_a.img` (sha 32a0ba7c), which no longer matches the `-4-3` descriptors in
  `vbmeta_a` (0e748586). The gate does validate image↔vbmeta descriptors correctly.
  Lesson: always flash from **live slot B copies**, never the pre-slot-A-copy backup images.
  `vbmeta_system_a`, `dtbo_a`, `init_boot_a`, `vendor_boot_a` flashed OK, but `vbmeta_system_a`
  and `dtbo_a`/`vendor_boot_a` were also stale `-4-2` backups → that is the incident below.
  fastbootd works (`is-userspace=yes`, `current-slot=a`, `partition-size:system_a=0x28F26000`).
- **INCIDENT**: the dry run mistakenly flashed the *pre-slot-A-copy backup* images for
  `vbmeta_system_a`, `dtbo_a`, `vendor_boot_a` — those were slot A's `-4-2` versions
  (vbmeta_system rollback index 5). Slot A therefore cannot boot and fell back; phone then dropped
  off USB entirely (no adb, no fastboot, no Motorola USB device). Build unaffected.
- Recovery prepared: `recover-slot-a.ps1` running in background — waits for the phone to reappear,
  then writes the correct `-4-3` `vbmeta_system_a`/`dtbo_a`/`vendor_boot_a` (slot B copies) via
  fastboot (or dd from rooted slot B), `--set-active=a`, reboots, verifies. Log:
  `recover-slot-a-2026-11-18.txt`.
- If the phone needs a physical wake: power-button press / USB replug is enough; no partitions
  are in a state that needs a BROM tool.

## 2026-11-18 14:11 UTC — PC reboot
- [owner]'s PC rebooted (server restart) at ~97% of the build (166,893/171,103 edges, `out/` 97 GB).
  Build did **not** fail or OOM; ninja state is intact.
- Resumed `m -j12 bacon` (incremental, packaging phase). WSL 23 GB RAM, 723 GB free.
- Recovery watchdog restarted (`recover-slot-a2.ps1`).
- Phone still absent from USB (no adb, no fastboot, no Motorola USB device).

## 2026-11-18 — build complete + validation
- Build finished **exit 0** (11:53 wall after resume). Images in `out/target/product/vegas26`.
- Staged to `MotoRoot\lineage-release\` (2.71 GB) with `SHA256SUMS`.
- Validation (AGENTS.md):
  - `vbmeta` rollback index **7**, flags 1 (verification disabled); `vbmeta_system` rollback **7**.
  - security patch props: boot/vendor `2026-06-01`, system/product/system_ext `2026-11-01` (>= stock).
  - boot kernel == stock `vegas26-kernels/5.15/Image.gz`; vendor_boot DTB == stock `mt6835.dtb`.
  - `dtbo.img` payload identical to stock; only the AVB footer/fingerprint prop differs (build re-signs
    the prebuilt dtbo with its own key) — expected.
  - `avbtool verify_image` passes for boot/init_boot/vendor_boot/dtbo.
  - build fingerprint: `motorola/lineage_vegas26/vegas26:16/BP4A.251205.006/eng.ferg:userdebug/test-keys`
    (test-keys; tree author tested release keys — watch for TEE trustlet issues on first boot).
- Phone recovery: watchdog flashed correct `-4-3` `vbmeta_system_a`/`dtbo_a`/`vendor_boot_a` at 07:26,
  slot A boots. Phone had been in Android **safe mode** (`mSafeMode=true`) after the bootloops and had
  airplane mode on; cleared airplane mode, Wi-Fi on, rebooted → `mSafeMode=false`, Wi-Fi enabled.
- Note: the earlier `dtbo`/`vendor_boot`/`vbmeta_system` flash tests proved fastboot flashing works when
  the image matches the device's `vbmeta` descriptors; `boot_a` only failed because the stale `-4-2`
  backup was used.

## 2026-11-18 — flash part 1 + USB trouble
- `flash-lineageos.ps1 -DoWipe -DoGapps` started. Boot chain flashed to slot A in bootloader fastboot:
  `vbmeta_a`, `vbmeta_system_a`, `boot_a`, `init_boot_a`, `vendor_boot_a`, `dtbo_a` — all OK.
- **Verified live on device (recovery adb, root):**
  - `vbmeta_a` first 64 KB = release `9fe3510b…` (partition is 8 MB; only 64 KB written — expected).
  - `vbmeta_system_a` first 64 KB = release `98afe43a…`.
  - `boot_a` = `d118b1da…`, `init_boot_a` = `a686dcbe…`, `vendor_boot_a` = `8294b25d…`,
    `dtbo_a` = `ee983052…` — all byte-identical to the release images.
- USB: fastbootd interface (`VID_18D1&PID_4EE0`) came up as `CM_PROB_FAILED_INSTALL`; Zadig bound
  WinUSB to the ADB interface (`PID_D001`, "ADB Device"). fastboot still cannot see the device.
- Recovery adb: [owner] tapped **Advanced → Enable ADB**; `adb root` works in Lineage Recovery
  (uid 0), so block devices are readable for verification.
- `adb reboot bootloader` from recovery → phone drops off USB entirely (no VID_18D1 node).
  [owner] replugged the cable; node reappears only after replug.
- [owner] reports the phone is now on a **Lineage setup screen** (system appears to be booting).
- Fallback super.img: `m bacon superimage` still running as an orphaned ninja process (25%,
  ~2h left). `m superimage-nodeps` could not run (out/.lock held by that build).
- Watcher armed: `usb-watch.ps1` → `usb-watch.log` (10 min, logs USB/adb/fastboot state changes).

## 2026-11-18 — LineageOS first boot CONFIRMED
- [owner] reports the phone shows **"Welcome to LineageOS"** → the logical partitions were flashed and
  the system booted. Flash part 2 therefore did complete (wipe-super + 5 logical images + `-w`).
- MindTheGapps sideload status still unknown (adb was unauthorized at that point in the script);
  verify once adb returns.
- USB still not enumerating while LineageOS runs (no VID_18D1 node) → cable replug needed.

## 2026-11-18 — LineageOS verified + Gapps attempt
- USB came back after [owner] enabled USB debugging; `adb` reported `[serial redacted] device
  product:lineage_vegas26 model:moto_g_power___2026`.
- Verified on device: `ro.lineage.version = 23.2-20261118-UNOFFICIAL-vegas26`,
  `ro.build.display.id = BP4A.251205.006 test-keys`, `ro.build.type = userdebug`,
  slot `_a`, security patch `2026-11-01`, kernel `5.15.189-android13-8-…`, `flash.locked=0`,
  verifiedboot `orange`. `ro.build.fingerprint` is Motorola's stock string — the device tree
  overrides the fingerprint on purpose; `ro.lineage.*` props confirm our build.
- Wi-Fi confirmed working: `cmd wifi status` → connected to [wifi-ssid], 11ac, 390 Mbps,
  RSSI -53, IP [local-ip]. [owner] confirmed Wi-Fi works.
- MindTheGapps: `g.prop` ABSENT, no `com.google.android.gms`/`com.android.vending`.
  First `adb sideload` attempt stalled at ~46% and was killed by a 15 min tool timeout;
  a retry returned instantly with `Total xfer: 0.00x` (exit 0) — recovery reported a
  fuse write/timeout error, [owner] restarted "Apply update → Apply from ADB".
  Retry running in background → `gapps-sideload2.log`.
- Upstream re-check (17:5x UTC): all 8 repos still at the same `lineage-23.2` HEAD as
  GitHub — no new commits/releases/tags.
- Magisk APK present locally: `MotoRoot\Magisk-v31.1.apk` (for re-rooting `init_boot_a`).

## 2026-11-20 — sanitized public GitHub handoff packet
- Built `public-repo\` staging folder with only 4 sanitized markdown docs (README.md,
  HOWTO-vegas26-root-lineage.md, ACTIONS-LOG.md, AGENT-HANDOFF.md). Excluded all backups,
  APKs, firmware images, keys, and raw logs.
- Redacted: device serial, unlock secret/key, local Windows paths, owner name, Wi-Fi
  SSID/IP/BSSID. Verified zero sensitive-token matches before and after upload.
- Initialized git on `main`; first commit `c4e376f` ("Add sanitized vegas26 unlock/root/LineageOS
  handoff").
- Created public repo `vegas26-lineageos-handoff` on the owner's GitHub account and pushed `main`.
- Verified remote: PUBLIC visibility, all 4 files present, zero redaction hits in the fetched
  remote content.
- This log entry appended afterward; public copy regenerated and re-pushed.

## 2026-11-20 — carrier/SIM unlock attempt (modem patch, slot A only)
- Clarified that bootloader unlock and carrier unlock are separate: the SIM lock lives in the modem
  (`md1img`), not Android or the bootloader. Official Straight Talk/TracFone policy requires 365
  paid active days (activated on/after 2026-01-20), so this phone is not eligible.
- Read-only compatibility check with `andrewwm/vegas-freedom` (HEAD `ed78c7e`) on the verified
  backup `md1img_a.img` (sha 432096bc…, trimmed dab9e69e…): **COMPATIBLE (VERIFIED-LIVE XT2617V
  -4-2)**, all 8 SIM-lock signatures at expected offsets. The -4-3 modem is byte-identical, so no
  `--experimental-relocate` was needed.
- Built locally: `md1img_vegas_patched.signed.img` sha `f6bd718ffb5c734cdab9e3c9c8c83987bf8e7b71299adb372cf678940b27046a`
  (exact match to the tool's verified-live image). Revert image `md1img_stock_trimmed.img` staged.
- Preflight in bootloader: `securestate=flashing_unlocked`, `current-slot=a`, `is-userspace=no`,
  bootloader `MBM-3.1-…-W1WEOS36.48-22-3-3-4-3-7806ed`. (First two script attempts aborted on my
  getvar parser before any write; third attempt passed.)
- **Flashed `md1img_a` ONLY** — `OKAY`, 74305 KB. Rebooted; slot `_a`, `sys.boot_completed=1`,
  baseband unchanged, `ro.lineage.version` unchanged. No SIM inserted, so `gsm.sim.state` still
  `ABSENT,NOT_READY` — **functional unlock UNVERIFIED**.
- `adb root` was initially disabled on the running LineageOS (Developer options toggle required for
  readback verification). Slot B remains untouched stock `-4-3` rollback lever. Revert path: reflash
  `work/modem_work/md1img_stock_trimmed.img` to `md1img_a`.
- Readback verification (after [owner] enabled rooted debugging): `adb root` → uid 0, dumped live
  `/dev/block/by-name/md1img_a`, and hashed the first 76,088,448 bytes both on-device and locally
  after `adb pull`. Both = `f6bd718ffb5c734cdab9e3c9c8c83987bf8e7b71299adb372cf678940b27046a`,
  an exact match to the patched image. Flash confirmed byte-for-byte. Full partition hash changed
  from backup `432096bc…` to `7eb85818…` as expected. SIM still absent, so carrier acceptance is
  still untested.
