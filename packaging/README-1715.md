# viogpudo @DRV_VERSION@ (PR #1715, HWCursor=1) — test-signed PoC package (x64)

virtio-gpu display-only driver (viogpudo) built from upstream pull request
[virtio-win/kvm-guest-drivers-windows#1715](https://github.com/virtio-win/kvm-guest-drivers-windows/pull/1715),
head `74594634`, driver source unchanged. **Test-signed — not for production.**

The PR is the reviewed successor of `eaee2b2` (tested as 100.200.977.1 / 977.4). Compared with
`eaee2b2` it:

- allocates the cursor-upload wait event from non-paged pool instead of the stack (safe if the
  host answers after the 100 ms timeout);
- reads `auto_release` before calling a control-queue completion callback (`DpcRoutine`);
- clears the whole 64x64 cursor resource and uploads all of it on every shape change, so no pixels
  of a previous, larger shape stay visible.

`viogpudo.inf` differs from the upstream build output in one line: `HKR,, HWCursor, %REG_DWORD%, 1`
(upstream: 0), so the hardware cursor is on after install. It is written on every install of this
package onto the device; a manual change is reset to 1 on the next (re)install. The driver reads it
when it starts.

The catalog `viogpudo.cat` covers every platform below and is signed with the same test
certificate as the 977.x packages.

## Platforms (x64 only)

| Platform | Build | Notes |
|---|---|---|
| Windows 10 | 1607 (14393) and later | |
| Windows 11 | all | check Memory Integrity, see below |
| Windows Server 2016 | 14393 | Desktop Experience |
| Windows Server 2019 | 17763 | Desktop Experience |
| Windows Server 2022 | 20348 | Desktop Experience |
| Windows Server 2025 | 26100 | Desktop Experience; check Memory Integrity |

This build has not been tested yet on any platform.

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

The certificate and test-signing steps can be skipped on a VM that already runs a 977.x package.

Then:

1. Reboot.
2. Check the display adapter's driver version is **@DRV_VERSION@** (Device Manager → Display
   adapters → Driver). If Windows kept the old driver: Update driver → Browse → Let me pick →
   Have Disk → this `viogpudo.inf`.

## What to check

1. Hover over text and window edges: a single I-beam and single resize arrows through the hardware
   cursor, no echo and no stale previous shape (same as 977.x).
2. New in this build: change from a large pointer to a smaller one (for example the busy/working
   pointer to the I-beam or a resize arrow) and look for leftover pixels of the previous shape
   around the new one.
3. No hang, freeze or bugcheck during sustained pointer activity (moving, many shape changes,
   dragging windows).

With the 977.x builds a second, lagging arrow was visible over noVNC while dragging or resizing a
window (measured once on Windows 10). That is not a failure criterion here; just note whether it
still happens.

## Rollback

```
pnputil /enum-drivers                      (find the oemNN.inf for viogpudo @DRV_VERSION@)
pnputil /delete-driver oemNN.inf /uninstall
bcdedit /set testsigning off
shutdown /r /t 0
```

## Files

`viogpudo.sys`, `viogpudo.inf`, `viogpudo.cat`, `viogpudo.pdb` (symbols),
`ClassroomPocDriver.cer` (public test certificate), `BUILD-INFO.txt`, `SHA256SUMS.txt`.
