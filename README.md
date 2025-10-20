# win11-upgrade-offline-playbook
# Windows 11 in‑place upgrade – stabil „offline” playbook

Cél: átvinni a Windows 10 → Windows 11 helyben frissítést olyan gépeken, ahol a telepítő a SAFE_OS / APPLY_IMAGE fázisban elakad (gyakran driver vagy EFI okok miatt).

Kulcsszavak: Windows 11 upgrade, APPLY_IMAGE, SAFE_OS, Dynamic Update, AMD RAID, Intel RST, rcraid, rcbottom, rccfg, iaStor, AHCI, NVMe, CSM, UEFI, ESP, EFI size, offline setup

## Rövid lényeg
- A hálózat teljes lekapcsolása (LAN/Wi‑Fi) megakadályozza, hogy a telepítő „Dynamic Update” néven problémás drivereket töltsön le és injektáljon.
- RAID/RST maradványok eltávolítása és UEFI/AHCI/CSM helyes beállítása kritikus.
- Kis (~100 MB) EFI/ESP partíció gond lehet; ajánlott ≥260–300 MB és ≥50–80 MB szabad hely.

## Beváltt lépések

1) BIOS/UEFI
- UEFI boot, CSM Disabled
- SATA Mode = AHCI
- fTPM/TPM 2.0 Enabled
- Boot Option #1 = Windows Boot Manager

2) Tároló driverek tisztítása (Admin PowerShell/CMD)
```cmd
pnputil /enum-drivers | findstr /i "iaahci iastor rcraid rcbottom rccfg"
pnputil /delete-driver oemXX.inf /uninstall /force
sc config rcraid start= disabled
sc config rcbottom start= disabled
sc config rccfg start= disabled
```
- NVMe: a Microsoft „Standard NVM Express Controller” fusson.

3) Rendszerintegritás és takarítás
```cmd
sfc /scannow
DISM /Online /Cleanup-Image /RestoreHealth
chkdsk C: /scan
```
- Töröld: `C:\$WINDOWS.~BT`, `C:\$WINDOWS.~WS`
- Legyen 30–40 GB szabad hely.

4) EFI/ESP ellenőrzése
```cmd
mountvol S: /S
fsutil volume diskfree S:
```
- Ajánlott: ≥260–300 MB, ≥50–80 MB szabad.
- Ha csak bővíted a meglévő ESP-t: `bcdboot` nem kötelező.
- Ha ÚJ ESP-t hozol létre: `bcdboot C:\Windows /s S: /f UEFI`

5) OFFLINE telepítés (kritikus!)
- Húzd le a LAN kábelt és kapcsold ki a Wi‑Fi-t, VAGY:  
  `setup.exe /DynamicUpdate disable`
- ISO-ból: `setup.exe` → „Nem most” → „Személyes fájlok és alkalmazások megtartása”
- (Opcionális) WU szolgáltatások stop:  
  `net stop wuauserv` és `net stop bits`

6) Utóellenőrzések
```cmd
pnputil /enum-drivers | findstr /i "iaahci iastor rcraid rcbottom rccfg"
reagentc /info
```
- (Opcionális) Secure Boot bekapcsolása

## Miért segít az OFFLINE?
A Dynamic Update több szakaszban próbál friss komponenseket/illesztőket letölteni. Offline módban csak az ISO tartalma fut, így nem kerülhet vissza problémás RAID/RST driver.

## Esettanulmányok (dataset)
A `dataset/*.jsonl` fájlok gépileg feldolgozható esettanulmányok. Lásd: [dataset/README.md](dataset/README.md).

## License

This repository is licensed under the Creative Commons Attribution 4.0 International (CC BY 4.0). You are free to share and adapt, provided you give appropriate credit and indicate changes.

[Read the full license text](https://creativecommons.org/licenses/by/4.0/).
