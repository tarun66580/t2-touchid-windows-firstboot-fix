# Validation record

## Environment

- Windows 11 test machine on an Intel MacBook with Apple T2.
- Windows Hello had an enrolled fingerprint before the boot test.
- Active Touch ID package: `DriverVer 10/10/2026, 1.2026.1010.2145`.
- Test date: 2026-10-10.

## Observed boot order

1. The system booted at 22:26:47 local time.
2. `T2SepBootstrap` completed its SEP unlock sequence at 22:27:00.
3. Windows Biometric Service created the Apple T2 Touch ID unit shortly after
   that point.
4. Windows recorded a successful identification for the enrolled user at
   22:27:12, at the first sign-in screen after restart.

The biometric device reported `Status: OK` and problem code `0`.

## Regression checks

- The original private-sensor-type transfer failure (event 1114,
  `0x80098036`) was absent.
- Windows Biometric Service used the in-box sensor and storage adapters with
  `T2TouchIdEngineAdapter.dll` as the engine adapter.
- The system still retained PIN/password sign-in as a recovery path.

## Notes

The Windows Hello Face Virtual Software Device may emit unrelated events on
this test system. Those events identify the Face virtual device and are not
Apple T2 Touch ID failures.
