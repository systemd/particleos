---
title: Testing on particleos
category: Contributing
layout: default
SPDX-License-Identifier: LGPL-2.1-or-later
---

# Testing on particleos

Welcome to the **particleOS Testing and Validation Guide**. This document
outlines the comprehensive procedure for developers and contributors, covering
the entire workflow from building the particle OS image to performing an
installation on a simulated empty drive and finally testing the resulting
installed system within a Virtual Machine environment.

The goal is to provide clear, actionable instructions to facilitate a smooth
setup of your development environment, enabling effective building,
installation, and testing of changes to the particleOS project.

Welcome to the particleos testing guide! This document provides the step by step
guide from the particleOS image creation to the test in Virtual Machine
of the final installed image, going through the installation process on an
empty drive.

It aims to be a comprehensive guide for developers who want to contribute to the
project by providing clear instructions on how to set up a development
environment, build the project, and test their changes.

## Prerequisites

### Environment setup

* Before you start testing on particleos, make sure to have the **lastest**
version of `mkosi` accessible from your path-environment.

  ```bash
  cd ${WORKDIR}
  git clone https://github.com/systemd/mkosi.git
  export PATH="${WORKDIR}/mkosi/bin/:${PATH}"
  ```

* `mkosi` requires Python 3.8 or higher. You can install the lastest python
version and force the Interpreter as follow:

  ```bash
  export MKOSI_INTERPRETER="/usr/bin/python3.12"
  ```

* Verify the version of mkosi installed:

  ```bash
  mkosi --version
  ```

* You will also need `qemu` installed on your system to test the created images in
a virtual machine.

### Lastest systemd version (optional but recommended)

As documented in the main `README.md`, it is highly recommended to use the
latest version of `systemd`. Choose one of the following methods:

* [Using the OBS profile](../README.md#using-the-obs-profile-to-fetch-a-newer-systemd)
* [Building systemd from source locally](../README.md#building-systemd-from-source)

## Building the particleOS Image

To build the particleOS Image:

1. Navigate to the root of the particleos repository and create the local
  config `mkosi.local.conf` file with the
  following content:

  ```conf
  [Match]
  Distribution=opensuse

  [Distribution]
  Release=tumbleweed

  [Runtime]
  VirtualMachineMonitor=qemu

  QemuArgs=
          -drive if=none,file=./mkosi.output/target-disk.img,format=raw,id=installdisk
          -device virtio-blk-pci,drive=installdisk
  ```

  * Use the distribution and its release as your convenience. The list of
  supported distributions and releases can be found in the files:
  `mkosi.conf.d/${DISTRIBUTION}/mkosi.conf`.

  > Notes:
  >  * The `QemuArgs` section is used to pass an empty created disk image as a
  >    drive to the virtual machine. This allows you to test the installation
  >    process directly on the created image.
  >  * You may want to adjust the disk image memory according to your needs with
  >    `RAM=`, do not put less than 4G.

2. run the following commands to generate the particleOS image:

  ```bash
  mkosi genkey
  mkosi --debug -B -ff -d opensuse -r tumbleweed
  ```

  This command will generate artifacts, particleOS Image and place them in the
  `mkosi.output` directory.

  ```bash
  initrd
  initrd.cpio.zst
  ParticleOS_20251121163001_x86-64        # <--- Link to the ParticleOS Image
  ParticleOS_20251121163001_x86-64.efi
  ParticleOS_20251121163001_x86-64.esp.raw
  ParticleOS_20251121163001_x86-64.manifest
  ParticleOS_20251121163001_x86-64.raw    # <--- ParticleOS with installer
  ParticleOS_20251121163001_x86-64.usr-x86-64.ecb75f74bf45139cead2d7b715285f8d.raw
  ParticleOS_20251121163001_x86-64.usr-x86-64-verity.e5ee934fea3a94e51c5474e9a40eecf5.raw
  ParticleOS_20251121163001_x86-64.usr-x86-64-verity-sig.0068c49513da4b0ab1767da560f42289.raw
  ```

## Install the full image from the particleOS Image

To simulate the installation on a target from the particleOS Image, you can
create an empty image file that will act as the target disk for the
installation. Then, you can use `mkosi vm` to boot the particleOS Image and
perform the installation process.

* First, create an empty disk image file (e.g., 30GB):

  ```bash
  qemu-img create -f raw ./mkosi.output/target-disk.img 30G
  ```

* Next, boot the particleOS using `mkosi vm`, passing the empty disk
image as a drive (already done in the `mkosi.local.conf` file):

  ```bash
  mkosi vm
  ```

* When the boot menu appears, choose the entry "Live Image (Installer)" and wait
  for the system to boot
* It autologins as user `particleos` (password `particleos`). Log in as root to
  initiate the installation process:

  ```bash
  su -
  ```

  * Verify that the target disk is available:

  ```bash
  lsblk
  ```

  You should see the `vdb` disk representing the empty disk image created
* Start the installation process by running the following command in the terminal:

  ```bash
  systemd-repart --dry-run=no --empty=force --defer-partitions=swap,root,home /dev/vdb
  ```

* Press `Ctrl a + c` and type "quit" in the `qemu` console to turn the machine
  off.

## Testing the installed image (in console mode)

The easiest way to test the target disk image is to use again `mkosi vm`, to
re-use the already configured secureboot environment and TPM.

Due to current limitations with `systemd-repart`, see [systemd issue #283](https://github.com/systemd/systemd/issues/38907#issuecomment-3562322145),
it is preferable to boot only the `target-disk.img` without attaching the image
generated by particleos.

1. change the default link from mkosi.output to point to the
  `target-disk.img`:

  ```bash
  ln -sf target-disk.img mkosi.output/ParticleOS_current_x86-64
  ```

2. Removes lines related to the installation disk from the
  `mkosi.local.conf` file created previously:

  ```conf
  [Match]
  Distribution=opensuse

  [Distribution]
  Release=tumbleweed

  [Runtime]
  VirtualMachineMonitor=qemu
  #QemuArgs=
  #        -drive if=none,file=./mkosi.output/target-disk.img,format=raw,id=installdisk
  #        -device virtio-blk-pci,drive=installdisk
  ```

3. run the vm:

  ```
  mkosi vm
  ```

4. Boot as a regular image and run the following command to setting up the home
  dir as explained at [Configuring systemd-homed after installation](../README.md#configuring-systemd-homed-after-installation).
