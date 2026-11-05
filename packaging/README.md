# viogpudo 100.200.977.1 — test-signed PoC package (x64)

virtio-gpu display-only driver (viogpudo) from upstream `virtio-win/kvm-guest-drivers-windows`
commit `eaee2b2` (#977 cursor fixes: synchronous cursor-image upload, hardware cursor for
monochrome/masked shapes such as the I-beam). **Test-signed — not for production.**

`viogpudo.sys` and `viogpudo.inf` are byte-identical to the build tested on `clientw10`
(run 41664850081). Only the catalog `viogpudo.cat` was regenerated, listing every target
platform below, and signed with the same test certificate.

## Platforms (x64 only)

| Platform | Build | Notes |
|---|---|---|
| Windows 10 | 1607 (14393) and later | tested on 19041 (`clientw10`) |
| Windows 11 | all | check Memory Integrity, see below |
| Windows Server 2016 | 14393 | Desktop Experience |
| Windows Server 2019 | 17763 | Desktop Experience |
| Windows Server 2022 | 20348 | Desktop Experience |
| Windows Server 2025 | 26100 | Desktop Experience; check Memory Integrity |

Only Windows 10 has been tested so far. The others are expected to work (same NT 10.0 kernel
API, INF section `NTamd64.10.0`) but are unverified.

## Prerequisites per VM

- KubeVirt VM spec: `devices.video.type: virtio`
- **Secure Boot off** — otherwise `bcdedit /set testsigning on` is refused
- Memory Integrity (HVCI) may block the driver on Windows 11 / Server 2025. If the driver does
  not load, check Event Viewer → *Applications and Services Logs → Microsoft → Windows →
  CodeIntegrity → Operational*; turning Memory Integrity off is the test workaround.

## Install (elevated prompt)

```
certutil -addstore Root ClassroomPocDriver.cer
certutil -addstore TrustedPublisher ClassroomPocDriver.cer
bcdedit /set testsigning on
shutdown /r /t 0

pnputil /add-driver viogpudo.inf /install
```

Then:

1. Check the display adapter's driver version is **100.200.977.1** (Device Manager → Display
   adapters → Driver). If Windows kept the old driver: Update driver → Browse → Let me pick →
   Have Disk → this `viogpudo.inf`.
2. **Set `HWCursor` = 1 (DWORD)** under
   `HKLM\SYSTEM\CurrentControlSet\Control\Class\{4d36e968-e325-11ce-bfc1-08002be10318}\<NNNN>`
   (the subkey whose `DriverDesc` is the virtio GPU). The INF resets it to 0 on every install.
3. Reboot.

## Expected behaviour

- Hover shape changes (I-beam, resize arrows) via the hardware cursor, no echo.
- While dragging or resizing a window, a second, lagging arrow can be visible over QEMU VNC
  (noVNC) for the duration of the drag only. This is not a failure criterion for this package.
  What was measured (one drag, Windows 10, WPP trace plus noVNC pixel check): at the start of the
  move/size loop Windows hides the hardware cursor (`SetPointerPosition` with `Visible = 0`)
  without a failing driver call before it, and an arrow is drawn into the framebuffer while noVNC
  keeps showing its own cursor. Likely, but not verified: QEMU's VNC server does not pass the hide
  on to the client. Whether the driver could avoid it has not been established.

## Rollback

```
pnputil /enum-drivers                      (find the oemNN.inf for viogpudo 100.200.977.1)
pnputil /delete-driver oemNN.inf /uninstall
bcdedit /set testsigning off
shutdown /r /t 0
```

## Files

`viogpudo.sys`, `viogpudo.inf`, `viogpudo.cat`, `viogpudo.pdb` (symbols),
`ClassroomPocDriver.cer` (public test certificate), `BUILD-INFO.txt`, `SHA256SUMS.txt`.
