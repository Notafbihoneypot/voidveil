# Voidveil

Voidveil is an experimental, console-only, amnesic live system based on
Void Linux, designed for Qubes HVM testing with fail-closed Tor networking.

> [!WARNING]
> This is a research prototype. It has not been independently audited and is
> not equivalent to Tails. Do not use it with secrets. Test only in a
> disposable Qubes HVM without attached physical disks, USB controllers, or
> PCI devices.

## Current checkpoint

- Version: `0.1.0-alpha.3-qubes`
- Architecture: `x86_64-glibc`
- Image date: `2026-08-24`
- Pinned void-mklive commit:
  `e50f853db68d9910e8c8a1e82cde14d61c9a6fef`
- Expected ISO SHA-256:
  `17c9c1c09f83c3d980e9ac2104606756b7a1bd101c3f78ff666d208b65d59054`

The validated checkpoint passed QEMU TCG boot, fail-closed routing, Qubes
online static-route, Qubes offline zero-packet, read-only decoy-disk, and
BIOS/UEFI structure tests. It has not yet been tested on Xen/Qubes hardware.

Source and automated release workflow will be added before publishing a
download.
