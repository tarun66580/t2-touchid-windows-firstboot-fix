# Apple T2 Touch ID for Windows: first-boot fix

This repository contains the focused patch set and validation scripts used to
make Apple T2 Touch ID work at the **first Windows Hello sign-in after a
restart** on an Intel MacBook with a T2 chip.

It is an add-on for
[Osaul4ik/apple-t2-touchid-windows](https://github.com/Osaul4ik/apple-t2-touchid-windows),
reviewed against upstream revision
`df739878f2d6d1610d4620d228ce34e356d1e3e6`.

## Result

The tested Windows 11 system successfully identified the enrolled user at the
first lock screen after reboot. The active biometric driver was
`1.2026.1010.2145`, the device status was `OK`, and Windows Biometric Service
recorded a successful identification event. The previous Apple T2 unit startup
error, event 1114 with `0x80098036`, did not recur.

## What changed

### 1. WBDI private sensor-type response

`patches/0001-wbdi-private-sensor-type.patch` implements
`IOCTL_BIOMETRIC_GET_PRIVATE_SENSOR_TYPE` in the UMDF WBDI driver. The device
does not provide a private sensor-type blob, but it now returns a well-formed,
empty success payload instead of falling through to `STATUS_NOT_SUPPORTED`.

This removed the Windows Biometric Framework configuration failure recorded as
event 1114 / `0x80098036`.

### 2. Reliable service ordering at boot

`scripts/Set-T2TouchIdWbioStartupDependency.ps1` adds `T2SepBootstrap` to
Windows Biometric Service (`WbioSrvc`) dependencies. The bootstrap service
does not report `Running` until the SEP unlock sequence has finished, so WBF
does not create its biometric unit until the Touch ID side is ready.

The script preserves the existing dependency list, writes a rollback JSON file
before making a change, checks for dependency cycles, and provides `-Action
Undo` for rollback.

### 3. Evidence collection

`scripts/Collect-T2TouchIdFirstBootEvidence.ps1` captures the device version
and health, service configuration, Windows Biometric events, and the SEP log
after a boot test. It turns off the temporary Analytic event channel after
collection.

## Applying the patch

Start from the stated upstream revision and verify the target source before
applying the patch:

```powershell
git apply --check patches/0001-wbdi-private-sensor-type.patch
git apply patches/0001-wbdi-private-sensor-type.patch
```

Build and sign the modified driver locally using your own test-signing setup.
No signed driver, certificate, private key, SEP vault, macOS password, Windows
Hello template, or hardware log is distributed by this repository.

After the reviewed driver package is installed and Windows Hello enrollment is
known to work, run the service-ordering script from an elevated PowerShell
window:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\scripts\Set-T2TouchIdWbioStartupDependency.ps1
```

Restart Windows and test the enrolled finger at the first sign-in screen.
Then collect evidence:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\scripts\Collect-T2TouchIdFirstBootEvidence.ps1
```

## Rollback

The ordering script prints its rollback JSON path. To restore the original
Windows Biometric Service dependency list:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\scripts\Set-T2TouchIdWbioStartupDependency.ps1 -Action Undo -BackupPath "<printed rollback JSON path>"
```

Restart Windows once after rollback. Keep PIN or password sign-in available
while testing.

## Scope and safety

- Research code for Intel Macs with an Apple T2 chip running Windows.
- The software interacts with the T2 Secure Enclave. Use a machine that can
  be recovered and keep a macOS recovery path.
- This repository contains source-level changes and diagnostic scripts only.
  It intentionally excludes compiled drivers, catalogs, test certificates,
  private keys, user fingerprints, SEP vault data, event logs, and backups.
- The project is GPL-2.0-only. See [LICENSE](LICENSE) and [NOTICE](NOTICE).

## References

- [Windows Biometric Driver Interface](https://learn.microsoft.com/en-us/windows-hardware/drivers/biometric/biometric-driver-interface)
- [Biometric IOCTL calling sequence](https://learn.microsoft.com/en-us/windows-hardware/drivers/biometric/supporting-biometric-ioctl-calling-sequence)
- [Automatically starting services](https://learn.microsoft.com/en-us/windows/win32/services/automatically-starting-services)
