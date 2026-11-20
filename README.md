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

## Safety note

This is device- and firmware-specific. There is no known BROM rescue path for this handset — a
damaged bootloader chain can permanently brick it. Do not adapt these steps to another model.
Slot B was deliberately kept as a stock rollback lever (`fastboot --set-active=b`) and must not be
flashed. Validate live state with `adb` / `fastboot` before any write.
