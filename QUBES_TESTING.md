# Testing Voidveil in Qubes OS

Voidveil's Qubes image is a live ISO for an isolated HVM. It is not a Qubes
template and does not install Qubes agents. Start offline, verify the boot
audit, and only then attach a NetVM for the Tor leak test.

> [!WARNING]
> This alpha has not been independently audited. Keep the test qube free of
> secrets, do not attach USB or PCI devices, and do not use it for high-risk
> activity.

## 1. Verify the image outside dom0

Keep the ISO in a dedicated downloader/storage qube. Verify its checksum there:

```sh
sha256sum -c voidveil-0.1.0-alpha.3-qubes-x86_64-YYYYMMDD.iso.sha256
```

Do not copy the ISO into dom0. Qubes can expose an ISO stored in another qube
as a virtual CD-ROM.

## 2. Create an offline HVM

Run these commands in a dom0 terminal, replacing `iso-vm` and the path with the
qube and path that hold the ISO:

```sh
qvm-create voidveil-test \
  --class StandaloneVM \
  --property virt_mode=hvm \
  --property kernel='' \
  --label=orange
qvm-prefs voidveil-test memory 2048
qvm-prefs voidveil-test maxmem 2048
qvm-prefs voidveil-test netvm ''
qvm-prefs voidveil-test autostart false
qvm-start voidveil-test \
  --cdrom=iso-vm:/home/user/voidveil-0.1.0-alpha.3-qubes-x86_64-YYYYMMDD.iso
```

Select **Voidveil Qubes HVM (offline)**. The console must show
`VOIDVEIL-BOOT-AUDIT: PASS`. Inside the guest, `voidveil-audit` prints the full
report. NetworkManager and Tor deliberately remain stopped in this entry.

## 3. Enable the Qubes static network profile

Attach a clearnet NetVM and read the values Qubes assigned to the test qube:

```sh
qvm-prefs voidveil-test netvm sys-firewall
qvm-ls -n voidveil-test
qvm-prefs voidveil-test mac
```

Start from the ISO again. In the boot menu select
**Voidveil Qubes HVM online (edit IP and gateway)**, then edit the kernel line:

- BIOS/Syslinux: press `Tab`.
- UEFI/GRUB: press `e`, edit the line beginning with `linux`, then press
  `Ctrl-x`.

Append the assigned values, for example:

```text
voidveil.qubes_ip=10.137.0.23 voidveil.qubes_gateway=10.138.8.1
```

The menu entry already supplies `voidveil.qubes=1`. Voidveil validates both
addresses, creates a RAM-only NetworkManager profile with a `/32` guest address,
adds an on-link route to the Qubes gateway, and preserves the virtual MAC. If
either value is absent or malformed, the boot is marked unsafe and networking
remains disabled.

Use `sys-firewall`, not `sys-whonix`, for this test: Voidveil runs its own Tor
client, and Tor-over-Tor is not the intended path.

## 4. Check the guest

After the boot audit passes, inspect:

```sh
voidveil-audit
ip -4 address
ip -4 route
sv status NetworkManager tor
```

Expected properties:

- the assigned Qubes IP is present with a `/32` prefix;
- the Qubes gateway has a link route and is the default gateway;
- the interface retains the MAC shown by `qvm-prefs`;
- NetworkManager and Tor are running only after the audit passes;
- ordinary user TCP and DNS are redirected to Tor, while arbitrary UDP, LAN,
  IPv6, and direct non-Tor connections remain blocked.

## 5. Nested automated leak test (optional)

For the strongest repeatable test without changing the HVM guest, run QEMU
inside a dedicated Fedora StandaloneVM with virtualization extensions exposed
if your hardware and Qubes policy permit it. Install QEMU, xorriso, and tcpdump,
then run from the extracted source tree:

```sh
make qubes-leak-test ISO=/path/to/voidveil.iso
```

The target uses a 600-second default because nested emulation is slow. It boots
with a writable decoy disk, captures the virtual NIC, and fails if any reserved
test destination leaves the guest directly.

To exercise the Qubes online profile against QEMU's equivalent static network,
run:

```sh
VOIDVEIL_QEMU_APPEND='voidveil.qubes=1 voidveil.qubes_ip=10.0.2.15 voidveil.qubes_gateway=10.0.2.2' \
  make qubes-leak-test ISO=/path/to/voidveil.iso
```

## 6. Remove the test qube

When you no longer need the test environment, shut it down and remove it from
dom0:

```sh
qvm-shutdown --wait voidveil-test
qvm-remove voidveil-test
```

The live guest does not intentionally persist state, but deleting the test qube
also removes its Qubes-managed virtual disks and configuration.
