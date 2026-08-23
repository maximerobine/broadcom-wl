> [!IMPORTANT]
> This driver is no longer maintained.
> Please refer to the software packages referenced at the bottom of this page for up-to-date alternatives.

# Broadcom Linux hybrid wireless driver (64-bit)

Re-upload from the source code found on the [Broadcom Support and Downloads page][1]

**Patched for Linux >= 4.7**

Tested on a BCM4360-based 802.11ac Wireless Network Adapter (TP-LINK Archer T8E)

[1]: https://www.broadcom.com/support/download-search?pg=Wireless+Embedded+Solutions+and+RF+Components&pf=Legacy+Wireless&pa=Driver&dk=BCM4312&l=true

## Prerequisites

The following kernel modules are incompatible with this driver and should not be loaded:
* ssb
* bcma
* b43
* brcmsmac

Make sure to unload (`rmmod` command) and blacklist those modules in order to prevent them from being automatically
reloaded during the next system startup:

`/etc/modprobe.d/blacklist.conf`
```
# wireless drivers (conflict with Broadcom hybrid wireless driver 'wl')
blacklist ssb
blacklist bcma
blacklist b43
blacklist brcmsmac
```

## Compile and install

### Manually

Build and install for the running kernel:

```sh
$ make
$ make install
$ depmod -A
$ modprobe wl
```

### Automatically

Using [DKMS][2] and the included `dkms.conf` file, one can let the operating system rebuild and install the module
automatically on every new kernel installation:

```sh
$ dkms add /path/to/this/repo
$ dkms status
broadcom-wl, 6.30.223.271: added
```

Providing that the `dkms` service is enabled, the module should appear as *installed* in the list of modules managed by
DKMS after the system boots for the first time on the new kernel:

```sh
$ dkms status
broadcom-wl, 6.30.223.271, 4.7.6-200.x86_64, x86_64: installed
```

[2]: http://linux.dell.com/dkms/manpage.html

### Refreshing an existing DKMS registration

`dkms add` copies the source tree into `/usr/src/<package>-<version>` at the time it is run — pulling newer commits
into this repo (e.g. a kernel-compatibility fix) does **not** update that copy. If a kernel package upgrade triggers
a DKMS autobuild against stale sources, the build can fail and leave the kernel package (and `apt`/`dpkg` in
general) stuck half-configured. To refresh the registration with the current source:

```sh
$ sudo dkms remove -m broadcom-wl -v 6.30.223.271 --all
$ sudo dkms add /path/to/this/repo
$ sudo dkms autoinstall
```

If `apt`/`dpkg` was left in a broken state by a failed autobuild, finish configuring the interrupted packages
afterwards:

```sh
$ sudo dpkg --configure -a
```

## Bluetooth on combo cards (BCM43142)

This driver covers the WiFi half of the chip only. On combo parts such as the BCM43142 (`14e4:4365`), which pairs
802.11b/g/n with Bluetooth 4.0, the Bluetooth radio is a separate USB function (`0a5c:216c`) driven by the in-tree
`btusb`/`btbcm` modules — no out-of-tree module and no DKMS involved. It does however need a proprietary patch-RAM
blob that `linux-firmware` does not ship:

```
Bluetooth: hci0: BCM43142A0 (001.001.011) build 0000
Bluetooth: hci0: BCM: firmware Patch file not found, tried:
Bluetooth: hci0: BCM: 'brcm/BCM43142A0-0a5c-216c.hcd'
Bluetooth: hci0: BCM: 'brcm/BCM-0a5c-216c.hcd'
```

Without the patch the chip still runs its bare ROM, which makes the symptoms misleading: `btusb` binds, `hci0` comes
up `UP RUNNING`, `rfkill` is unblocked and `bluetoothd` reports a healthy controller — yet inquiry returns nothing and
no device is ever discovered. The `build 0000` line above, and an HCI revision of `0x0000`, are the tell.

The `.hcd` blobs can be extracted from the vendor's Windows driver package, or taken from
[winterheart/broadcom-bt-firmware][10], which performs that extraction mechanically. Install the file matching the USB
ID and reload the module:

```sh
$ sudo install -o root -g root -m 0644 BCM43142A0-0a5c-216c.hcd /lib/firmware/brcm/
$ sudo modprobe -r btusb && sudo modprobe btusb
```

A successful patch load is visible in the kernel log, with a non-zero build number:

```
Bluetooth: hci0: BCM43142A0 'brcm/BCM43142A0-0a5c-216c.hcd' Patch
Bluetooth: hci0: BCM43142A0 (001.001.011) build 0328
```

The blob lives outside any package, so it survives both reboots and `linux-firmware` upgrades and — unlike `wl` —
needs no rebuilding across kernel versions.

[10]: https://github.com/winterheart/broadcom-bt-firmware

## See also

* [Official README file][3] (download)
* Arch Linux packages: [broadcom-wl][4] / [broadcom-wl-dkms][5]
* Debian packages: [broadcom-sta][6] ([source repository][7])
* [kmod-wl][8] package for RPM Fusion ([source repository][9])

[3]: https://docs.broadcom.com/docs-and-downloads/docs/linux_sta/README_6.30.223.271.txt
[4]: https://archlinux.org/packages/extra/x86_64/broadcom-wl/
[5]: https://archlinux.org/packages/extra/x86_64/broadcom-wl-dkms/
[6]: https://packages.debian.org/source/sid/broadcom-sta
[7]: https://salsa.debian.org/broadcom-sta-team/broadcom-sta
[8]: http://download1.rpmfusion.org/nonfree/fedora/development/rawhide/Everything/x86_64/os/repoview/kmod-wl.html
[9]: https://pkgs.rpmfusion.org/cgit/nonfree/wl-kmod.git/
