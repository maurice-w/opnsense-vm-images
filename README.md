Building OPNsense 25.7 virtual machine disk images
==================================================

This fork of [opnsense/tools](https://github.com/opnsense/tools) adds a brief tutorial and two devices, `AMD64VM` and `ARM64VM`.
These devices allow building [OPNsense](https://opnsense.org/) VM images (amd64 and aarch64) with the console preset to EFI or serial instead of the default VGA console (System: Settings: Administration: Console).

This tutorial explains how to set up a virtualized build system from scratch and build a disk image.
Details about all build steps and options can be found in the official [opnsense/tools README](https://github.com/opnsense/tools/blob/master/README.md).

Sample VM images of major releases are published twice a year in the [GitHub releases](https://github.com/maurice-w/opnsense-vm-images/releases) section of this repository.
Up-to-date images (updated about every two weeks, following the official update schedule) as well as custom images are [available for sponsors](https://github.com/sponsors/maurice-w).

Setting up a build system
=========================

Download a [FreeBSD-14.3-RELEASE VM image](https://download.freebsd.org/releases/VM-IMAGES/14.3-RELEASE/) matching the CPU architecture (amd64 or aarch64) and hypervisor.
Extract the disk image and expand its size to at least 60 GB (Hyper-V: `Resize-VHD`, QEMU: `qemu-img resize`).
Attach the image to a VM, boot it and log in as root using the console (SSH is disabled by default).

Set a root password, enable SSH, update FreeBSD and reboot:

    # passwd
    # echo 'sshd_enable="YES"' >> /etc/rc.conf
    # echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config
    # freebsd-update fetch install
    # shutdown -r now

The remaining steps can be performed via SSH or the local console.

Install Git and clone this repository:

    # pkg install -y git
    # git clone https://github.com/maurice-w/opnsense-vm-images /usr/tools

Update all OPNsense code repositories:
    
    # cd /usr/tools
    # make update SETTINGS=25.7

Build and reinstall [pkg(8)](https://man.freebsd.org/cgi/man.cgi?query=pkg&sektion=8). This is required because the pkg version provided by FreeBSD is newer than the one used by OPNsense:

    # cd /usr/ports/ports-mgmt/pkg
    # make package
    # make reinstall
    # cd /usr/tools

Building a VM image
===================

The first argument of the `vm` command specifies the disk image format. All formats supported by [mkimg(1)](https://man.freebsd.org/cgi/man.cgi?query=mkimg) are available.

All VM images have a GUID partition table (GPT) and an EFI system partition (ESP) and support UEFI boot. Images for amd64 additionally support legacy BIOS boot.

Root and swap partition sizes can be customized with the second and third argument of the `vm` command.
The minimum root partition size is 4 GB. The swap partition can be omitted by setting the third argument to `never`.

The fourth argument of the `vm` command specifies the default console (EFI or serial). For a VGA console, this argument must be omitted.

By default, the root partition uses the UFS file system. For a ZFS file system, the `ZFS=zpool` build option must be added.

Building from source
--------------------

All [tagged OPNsense versions](https://github.com/opnsense/core/tags) can be build by setting the `VERSION` option accordingly.

VHDX image (Hyper-V), 4 GB root partition, no swap partition, EFI console, OPNsense 25.7.1-amd64, UFS file system:

    # make update vm-vhdx,4G,never,efi SETTINGS=25.7 VERSION=25.7.1 DEVICE=AMD64VM

QCOW2 image (QEMU), 30 GB root partition, 2 GB swap partition, serial console, OPNsense 25.7-amd64, ZFS file system:

    # make update vm-qcow2,30G,2G,serial SETTINGS=25.7 VERSION=25.7 DEVICE=AMD64VM ZFS=zpool

QCOW2 image (QEMU), 16 GB root partition, no swap partition, EFI console, OPNsense 25.7.5-aarch64, UFS file system:

    # make update vm-qcow2,16G,never,efi SETTINGS=25.7 VERSION=25.7.5 DEVICE=ARM64VM

QCOW2 image (QEMU), 20 GB root partition, 1 GB swap partition, serial console, OPNsense 25.7.2-aarch64, UFS file system:

    # make update vm-qcow2,20G,1G,serial SETTINGS=25.7 VERSION=25.7.2 DEVICE=ARM64VM

Using prefetched sets
---------------------

This method is much faster, but requires pre-compiled base, kernel and packages sets. The `VERSION` option specifies which version of the sets to download.

VHDX image (Hyper-V), 8 GB root partition, no swap partition, EFI console, OPNsense 25.7.1-amd64, ZFS file system:

    # make update prefetch-base,kernel,packages vm-vhdx,8G,never,efi SETTINGS=25.7 VERSION=25.7.1 DEVICE=AMD64VM ZFS=zpool

QCOW2 image (QEMU), 40 GB root partition, 4 GB swap partition, VGA console, OPNsense 25.7-amd64, UFS file system:

    # make update prefetch-base,kernel,packages vm-qcow2,40G,4G SETTINGS=25.7 VERSION=25.7 DEVICE=AMD64VM

[Official packages sets](https://pkg.opnsense.org/FreeBSD:14:amd64/25.7/sets/) are only published for some releases. If no packages set is available for the desired
OPNsense version, one can be created by downloading the individual packages and adding them to a tar archive.

QCOW2 image (QEMU), 20 GB root partition, no swap partition, serial console, OPNsense 25.7.6-amd64, UFS file system:

    # make update prefetch-base,kernel SETTINGS=25.7 VERSION=25.7.6
    # pkg install -y rsync
    # rsync -vaz rsync://mirror.level66.network/opnsense-dist/FreeBSD:14:amd64/25.7/MINT/25.7.6 /tmp
    # tar -C /tmp/25.7.6/latest -cf /usr/local/opnsense/build/25.7/amd64/sets/packages-25.7.6-amd64.tar .
    # make vm-qcow2,20G,never,serial SETTINGS=25.7 DEVICE=AMD64VM

The official mirrors only offer amd64 sets. A custom mirror needs to be specified for prefetching aarch64 sets.

QCOW2 image (QEMU), 10 GB root partition, no swap partition, EFI console, OPNsense 25.7.4-aarch64, UFS file system:

    # make update prefetch-base,kernel,packages vm-qcow2,10G,never,efi SETTINGS=25.7 VERSION=25.7.4 DEVICE=ARM64VM MIRRORS=https://opnsense-update.walker.earth

QCOW2 image (QEMU), 5 GB root partition, no swap partition, serial console, OPNsense 25.7.2-aarch64, UFS file system:

    # make update prefetch-base,kernel,packages vm-qcow2,5G,never,serial SETTINGS=25.7 VERSION=25.7.2 DEVICE=ARM64VM MIRRORS=https://opnsense-update.walker.earth

Since some OPNsense releases do not update the base and kernel, not every packages set is accompanied by base and kernel sets with the same version.
If this is the case for the desired OPNsense version, base and kernel sets must be prefetched separatly.

QCOW2 image (QEMU), 40 GB root partition, no swap partition, EFI console, OPNsense 25.7.7-aarch64 (assuming it uses the same base and kernel as 25.7.6), UFS file system:

    # make prefetch-base,kernel SETTINGS=25.7 VERSION=25.7.6 MIRRORS=https://opnsense-update.walker.earth
    # make update prefetch-packages vm-qcow2,40G,never,efi SETTINGS=25.7 VERSION=25.7.7 DEVICE=ARM64VM MIRRORS=https://opnsense-update.walker.earth

Downloading the VM image
========================

If successful, the VM image can be found under:

    # make print-IMAGESDIR

Download the image file:

    # scp root@[2001:db8::a]:/usr/local/opnsense/build/25.7/[amd64|aarch64]/images/OPNsense-25.7[.x]-vm-[amd64|aarch64].[qcow2|vhdx|...] .

Caveats
=======

- There are no official FreeBSD VHDX images. If setting up a build system in a Hyper-V Generation 2 VM, the VHD image must be converted to VHDX first (`Convert-VHD`).
- FreeBSD doesn't support Secure Boot, this needs to be disabled in the VM settings.
- The EFI / VGA console of the FreeBSD VM images defaults to a US keyboard layout. If required, this can be changed:

      # echo 'keymap="de.kbd"' >> /etc/rc.conf

- Out of the box, FreeBSD only supports SLAAC, not DHCPv6.
  [dhcpcd(8)](https://man.freebsd.org/cgi/man.cgi?query=dhcpcd) can be used if SLAAC isn't available:

      # pkg install -y dhcpcd
      # echo 'dhclient_program="/usr/local/sbin/dhcpcd"' >> /etc/rc.conf

  In IPv6-only networks without SLAAC, a temporary address needs to be added first to allow installing dhcpcd(8):

      # ifconfig [hn0|vtnet0] inet6 2001:db8::a

- Building OPNsense VHDX images is supported (`vm-vhdx`), but the VHDX file is as large as the specified partition sizes combined.
  Setting the root partition size to 4 GB and expanding the finished VHDX image to the desired size (`Resize-VHD`) is recommended.
