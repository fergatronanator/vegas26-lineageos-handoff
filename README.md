# vegas26 — unlock / root / LineageOS 23.2 handoff (sanitized)

Public, sanitized handoff packet for the Motorola Moto G Power 2026 (`XT2617V`, codename
`vegas26`) work: bootloader unlock, stock root + full backup, and a self-built LineageOS 23.2
install.

## What was redacted

- device serial number
- bootloader-unlock secret and key
- local Windows home paths
- Wi-Fi SSID / local IP / BSSID
- owner name
- private backup contents, APKs, firmware images, and raw logs (not included at all)

## Files

| File | Purpose |
|---|---|
| `HOWTO-vegas26-root-lineage.md` | Full step-by-step walkthrough (unlock → backup → root → build → flash) |
| `ACTIONS-LOG.md` | Append-only chronological log of every action and outcome |
| `AGENT-HANDOFF.md` | "Start here" document for another agent continuing the work |
| `CARRIER-UNLOCK.md` | Optional modem-level carrier/SIM unlock: provenance, what it changes, backups, verification, revert |

## Credits / upstream sources

This work stands on other people's research, code and threads. Credit and thanks to the following.
None of their binaries are bundled in this repo. The full credits block (with context for each) is
repeated in every document in this packet.

**Original guide and community thread**

- [parsimo2010/moto-g-power-2026-straight-talk-wifi-root-guide](https://github.com/parsimo2010/moto-g-power-2026-straight-talk-wifi-root-guide) — the original Wi-Fi-only setup / temp-root / bootloader-unlock / Magisk guide and guide kit (by u/parsimo).
- [Reddit r/StraightTalk announcement thread](https://www.reddit.com/r/StraightTalk/comments/1xwgm37/i_rooted_a_walmart_straight_talk_moto_g_power/) by u/parsimo. Community commenters: u/angel9990, u/mclcs, u/CORRUPT27, u/jghost1337, u/newofficemusic, u/No_Car_6879.

**Bootloader unlock, temporary root, Magisk**

- [Maikyxd/val-protocol](https://github.com/Maikyxd/val-protocol) — LK patch and unlock-key generation. Its own upstream credits: [kasnria001/pwnage24mtk](https://github.com/kasnria001/pwnage24mtk), [R0rt1z2/liblk](https://github.com/R0rt1z2/liblk), [R0rt1z2/lkpatcher](https://github.com/R0rt1z2/lkpatcher).
- [diabl0w/DFRoot](https://github.com/diabl0w/DFRoot) v4.7, [tiann/KernelSU](https://github.com/tiann/KernelSU) 3.3.1, [topjohnwu/Magisk](https://github.com/topjohnwu/Magisk) 31.1, [googlesamples/android-testdpc](https://github.com/googlesamples/android-testdpc).

**LineageOS 23.2**

- [jimmielin/android_device_motorola_vegas26](https://github.com/jimmielin/android_device_motorola_vegas26) (`lineage-23.2`) plus companion `jimmielin` and [LineageOS](https://github.com/LineageOS) trees; [MindTheGapps/16.0.0-arm64](https://github.com/MindTheGapps/16.0.0-arm64).
- Reference threads: [XDA UNOFFICIAL Lineage G Play 2026](https://xdaforums.com/t/unofficial-lineage-os-lineage-recovery-g-play-2026.4801191/) and [r/androidroot XT2615V unlock](https://www.reddit.com/r/androidroot/comments/1vkb0zl/success_unlocked_moto_g_play_2026_xt2615v/) by u/Significant-Noise530.

**Carrier / SIM unlock**

- [andrewwm/vegas-freedom](https://github.com/andrewwm/vegas-freedom) (`ed78c7e`), adapted from [Articfox291/kansas-modem-unlock](https://github.com/Articfox291/kansas-modem-unlock) and [crabcakes97/Valhalla](https://github.com/crabcakes97/Valhalla).

**Other background (reference only)**

- [jesussaves-dev/moto-g-power-2026-root](https://github.com/jesussaves-dev/moto-g-power-2026-root), [fuckyoumoto.xyz](https://fuckyoumoto.xyz); Google (Platform Tools) and Motorola (signed USB driver).

## Safety note

This is device- and firmware-specific. There is no known BROM rescue path for this handset — a
damaged bootloader chain can permanently brick it. Do not adapt these steps to another model.
Slot B was deliberately kept as a stock rollback lever (`fastboot --set-active=b`) and must not be
flashed. Validate live state with `adb` / `fastboot` before any write.
