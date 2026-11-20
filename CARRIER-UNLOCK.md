# Carrier / SIM unlock — vegas26

Sanitized companion to the main handoff. This covers the optional carrier-unlock work done **after**
the bootloader unlock, root and LineageOS install.

Key point: **bootloader unlock is not carrier unlock.** The SIM lock lives in the modem (`md1img`),
not in Android or the bootloader. Root and an unlocked bootloader are prerequisites for patching the
modem, but they do not remove the lock by themselves.

## Status at the time of writing

- Patched modem flashed to **slot A only**, byte-verified by readback.
- Functional carrier acceptance is **NOT yet tested** — no SIM was inserted. `gsm.sim.state`
  remained `ABSENT,NOT_READY`.
- Slot B remains untouched stock `-4-3` with Magisk; the rollback lever is intact.

## Official carrier route

Straight Talk / TracFone policy (effective 2026-07-31): devices activated on or after 2026-01-20
unlock on request after **365 cumulative days of paid, active service** on that specific device.
Military relocation orders are an exception. Eligibility is checked at `tfwunlockpolicy.com` or by
calling 1-888-442-5102. This device was set up Wi-Fi-only with no plan, so it is not eligible yet.

## What the patch does

- Patches **8 instruction sites** in the modem's SIM-lock enforcement code inside `md1img_a`:
  `custom_check_link_sml_legal_sim_rule`, `sml_sl_Check`, `smu_check_sml`, plus the gate checks.
- Re-signs the resulting image so the modem will load it.
- With a non-original-carrier SIM, `gsm.sim.state` should report `LOADED` instead of
  `NETWORK_LOCKED`. Android Settings may still display "locked" — that is carrier config/app state,
  not the modem.
- It does **not** change the IMEI, NVRAM, calibration data, baseband version string, bootloader,
  ROM, userdata or slot B.
- It does not guarantee every network will work: bands, VoLTE/VoWiFi and eSIM provisioning still
  vary by carrier.

## Provenance — where the firmware came from

No firmware was downloaded from any third party. The patched image was built locally from this
phone's own verified backup.

1. **Stock source:** `backups-2026-11-17/fullbackup/md1img_a.img`, 200 MB, SHA-256
   `432096bc67a109e8c979c718f417ec26fb1d27760eef27a4782d271e68bfe8c3`, verified against that
   backup's `sha256.csv`. Trimmed SHA-256
   `dab9e69e3bbb009dd2a7907f3603df101e15b11c3f34e1461745b1699c7d4757` — exactly the tool's
   documented known-good input hash. The `-4-3` modem is byte-identical to the validated `-4-2`
   profile, so no experimental relocate flag was needed.
2. **Patcher:** `https://github.com/andrewwm/vegas-freedom`, commit
   `ed78c7e5504341fbae2dfdfd0517c6248d367d59`. That repo contains scripts, a patch profile and a
   signer only — no proprietary firmware binaries, serial numbers or OEM keys. The modem-unlock
   code is credited as adapted from `Articfox291/kansas-modem-unlock`.
3. **Local build:** Python 3.12 with `capstone` + `pyasn1` produced
   `md1img_vegas_patched.signed.img`, SHA-256
   `f6bd718ffb5c734cdab9e3c9c8c83987bf8e7b71299adb372cf678940b27046a` — an exact match to the
   tool's published verified-live output hash.

## What was flashed

Preflight in bootloader: `securestate=flashing_unlocked`, `current-slot=a`, `is-userspace=no`,
bootloader `MBM-3.1-vegas_og_vext-5d518d-W1WEOS36.48-22-3-3-4-3-7806ed`.

```
fastboot flash md1img_a md1img_vegas_patched.signed.img    # OKAY, 74305 KB
fastboot reboot
```

After reboot: slot `_a`, `sys.boot_completed=1`, baseband string and ROM unchanged.

**Verification:** the live `/dev/block/by-name/md1img_a` was read back with root adb. The first
76,088,448 bytes hashed both on-device and locally to
`f6bd718ffb5c734cdab9e3c9c8c83987bf8e7b71299adb372cf678940b27046a` — an exact match to the patched
image. Full partition hash changed from `432096bc…` to `7eb85818…`, as expected.

## Backups (private — not included in this repository)

- `fullbackup/md1img_a.img` and `md1img_b.img` — 200 MB each, hashes verified against the manifest.
- `private-backups/` — `nvram`, `nvdata`, `persist`, `protect1`/`protect2`, `preloader_raw_a`/`_b`,
  `seccfg`, boot-chain images, and its own `sha256.csv`.
- Tool-generated revert image `md1img_stock_trimmed.img` — 76,088,352 bytes, SHA-256
  `dab9e69e3bbb009dd2a7907f3603df101e15b11c3f34e1461745b1699c7d4757`.
- Full partition backup: 94 images / 12.54 GB. Data backup: 52 tars / 14.22 GB.

Caveat: the full backup's **slot-A boot-chain images are stale `-4-2`** — do not reuse those for
flashing. The `md1img` images are fine.

## Revert

Reboot to bootloader and flash the stock trimmed image (or the original full 200 MB backup) back to
`md1img_a`, then reboot. For a full stock fallback, `fastboot --set-active=b` still boots the
untouched stock slot.

## Remaining test

Insert a non-original-carrier SIM or eSIM, reboot, then:

```
adb shell getprop gsm.sim.state
```

`LOADED` = success; `NETWORK_LOCKED` = failure. No SIM was present at the time of writing.
