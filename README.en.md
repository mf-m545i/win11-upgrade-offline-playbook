# Windows 11 in‑place upgrade — stable offline playbook

Goal: reliably complete Windows 10 → Windows 11 in‑place upgrades on machines that fail around the SAFE_OS / APPLY_IMAGE phase (often due to storage drivers or EFI constraints). This playbook documents an offline upgrade path, cleanup steps, and includes a small incident dataset (JSONL) for learning and reuse.

Keywords: Windows 11 upgrade, APPLY_IMAGE, SAFE_OS, Dynamic Update, AMD RAID, Intel RST, rcraid, rcbottom, rccfg, iaStor, AHCI, NVMe, CSM, UEFI, ESP, EFI size, offline setup, dataset

[Magyar verzió](README.md)

## TL;DR
- Fully offline install (disconnect LAN/Wi‑Fi, or run `setup.exe /DynamicUpdate disable`) prevents the installer from pulling problematic drivers during Dynamic Update.
- Remove RAID/RST remnants; ensure UEFI/AHCI/CSM are correctly configured.
- Small ESP can be tight; recommended ≥260–300 MB with ≥50–80 MB free.

## Steps (short version)

1) BIOS/UEFI
- UEFI boot, CSM Disabled
- SATA Mode = AHCI
- fTPM/TPM 2.0 Enabled
- Boot Option #1 = Windows Boot Manager

2) Storage drivers cleanup (Admin PowerShell/CMD)
```cmd
pnputil /enum-drivers | findstr /i "iaahci iastor rcraid rcbottom rccfg"
pnputil /delete-driver oemXX.inf /uninstall /force
sc config rcraid start= disabled
sc config rcbottom start= disabled
sc config rccfg start= disabled
```
- NVMe should use Microsoft “Standard NVM Express Controller”.

3) System integrity and cleanup
```cmd
sfc /scannow
DISM /Online /Cleanup-Image /RestoreHealth
chkdsk C: /scan
```
- Remove: `C:\$WINDOWS.~BT`, `C:\$WINDOWS.~WS`
- Ensure 30–40 GB free space.

4) Check ESP/EFI
```cmd
mountvol S: /S
fsutil volume diskfree S:
```
- Recommended: ≥260–300 MB, with ≥50–80 MB free.
- If you only resize the existing ESP: `bcdboot` is not required.
- If you create a NEW ESP: `bcdboot C:\Windows /s S: /f UEFI`

5) OFFLINE upgrade (critical)
- Disconnect LAN and turn off Wi‑Fi, OR:
```cmd
setup.exe /DynamicUpdate disable
```
- From ISO: `setup.exe` → “Not right now” → keep personal files and apps
- Optional: stop WU services
```cmd
net stop wuauserv
net stop bits
```

6) Post‑checks
```cmd
pnputil /enum-drivers | findstr /i "iaahci iastor rcraid rcbottom rccfg"
reagentc /info
```
- Optional: enable Secure Boot

## Why offline helps
Dynamic Update tries to fetch newer components/drivers during multiple phases. Offline mode uses only the ISO contents, preventing re‑introduction of problematic RAID/RST drivers.

## Incident dataset
Machine‑readable incidents live under `dataset/*.jsonl`. See the Hungarian README and `dataset/README.md` for the schema and an example (`incident-0001.jsonl`).

## License
Creative Commons Attribution 4.0 International (CC BY 4.0). You may share and adapt with proper attribution and indication of changes.
