# ParticleOS

ParticleOS is a fully customizable immutable distribution implementing the
concepts described in
[Fitting Everything Together](https://0pointer.net/blog/fitting-everything-together.html).

Note that ParticleOS is still in development, and we don't provide any backwards
compatibility guarantees at all.

The crucial difference that makes ParticleOS unique compared to other immutable
distributions is that users build the ParticleOS image themselves and sign it
with their own keys instead of installing vendor signed images. This allows
configuring the image to your liking by having full control over which
distribution is used as the base and which packages are installed into the
image.

The ParticleOS image is built using [mkosi](https://github.com/systemd/mkosi).
You will need to install the current main branch of mkosi to build current
ParticleOS images.

First, configure the variant you'd like to build in `mkosi.local.conf`. For a
desktop system, you'll want the `desktop` profile and either the `gnome` or the
`kde` profile.

```conf
[Distribution]
Distribution=arch

[Config]
Profiles=desktop,kde
```

To build the image, run `mkosi -B -f` from the ParticleOS repository. Currently
`arch`, `fedora` and `debian` are supported distributions. Implementing support for a
new distribution (that's already supported in mkosi) is as simple as writing the
necessary config files to install the required packages for that distribution.

To update the system after installation, you clone the ParticleOS repository
or your fork of it, make sure `mkosi.local.conf` is configured to your liking and
run `mkosi -B -ff sysupdate -- update --reboot` which will update the system using
`systemd-sysupdate` and then reboot.

## Using the OBS profile to fetch a newer systemd

Sometimes ParticleOS adopts systemd features as soon as they get merged into
systemd without waiting for an official release. That's why we recommend
enabling the `obs` profile to enable the systemd repositories on OBS
(https://software.opensuse.org//download.html?project=system%3Asystemd&package=systemd)
containing systemd packages which are built every day from systemd's git main
branch.

To enable the `obs` profile, add the following to `mkosi.local.conf`:

```conf
[Config]
Profiles=obs
```

## Building systemd from source

As an alternative to using the `obs` profile, you can build systemd from source:

```sh
git clone https://github.com/systemd/systemd
cd systemd
mkosi -f sandbox -- meson setup build
mkosi -f sandbox -- meson compile -C build
mkosi -t none -f
```

Then write the following to `mkosi.local.conf` in the ParticleOS repository to
use the artifacts from the systemd repository built by mkosi in ParticleOS:

```conf
[Content]
VolatilePackageDirectories=../systemd/build/mkosi.builddir/<distribution>~<release>~<arch>

[Build]
ExtraSearchPaths=../systemd/build
```

Make sure the distribution and release in `mkosi.local.conf` are identical in the
systemd checkout and the particleos checkout.

To build a newer systemd, run `git pull` in the systemd repository followed by
 `mkosi -f sandbox -- meson compile -C build` and `mkosi -t none`.

## Signing keys

ParticleOS images are signed for Secure Boot with the user's keys. To generate a new key,
run `mkosi genkey`. The key must be stored safely, it will be required to sign updates.

The key can be stored in a smartcard. Then you have to set the key in `mkosi.local.conf`:

```
[Validation]
SecureBootKey=pkcs11:object=Private key 1;type=private
SecureBootKeySource=provider:pkcs11
SignExpectedPcrKey=pkcs11:object=Private key 1;type=private
SignExpectedPcrKeySource=provider:pkcs11
VerityKey=pkcs11:object=Private key 1;type=private
VerityKeySource=provider:pkcs11
```

## Installation

Before installing ParticleOS, make sure that Secure Boot is in setup mode on the
target system. The Secure Boot mode can be configured in the UEFI firmware
interface of the target system. If there's an existing Linux installation on the
target system already, run `systemctl reboot --firmware-setup` to reboot into
the UEFI firmware interface. At the same time, make sure the UEFI firmware
interface is password protected so an attacker cannot just disable Secure Boot
again.

To install ParticleOS with a USB drive, first build the image on an existing
Linux system as described above. Then, burn it to the USB drive with
`mkosi burn /dev/<usb>`. Once burned to the USB drive, plug the USB drive into
the system onto which you'd like to install ParticleOS and boot into the USB
drive via the firmware. Then, boot into the "Installer" UKI profile. When you
end up in the root shell, run
`systemd-repart --dry-run=no --empty=force --defer-partitions=swap,root,home /dev/<drive>`
to install ParticleOS to the system's drive. Finally, reboot into the target
drive (not the USB) and the regular profile (not the installer one) to complete
the installation.

## LUKS recovery key

systemd doesn't support adding a recovery key to a partition enrolled with a token
only (tpm/fido2). It is possible to use cryptenroll to add a recovery password
to the root partition: `cryptsetup luksAddKey --token-type systemd-tpm2 /dev/<id>`

## Firmwares

Only firmwares that are dependencies of a kernel module are included, but some
modules don't declare their dependencies properly. Dependencies of a module can be
found with `modinfo`. If you experience missing firmwares, you should report
this to the module maintainer. `FirmwareInclude=` can be added in `mkosi.local.conf`
to include the firmware regardless of whether a module depends on it.

## Configuring systemd-homed after installation

After installing ParticleOS and logging into your systemd-homed managed user,
run the following to configure systemd-homed for the best experience:

```sh
homectl update \
    --auto-resize-mode=off \
    --disk-size=max \
    --luks-discard=on"
```

Disabling the auto resize mode avoids slow system boot and shutdown. Enabling
LUKS discard makes sure the home directory doesn't become inaccessible because
systemd-homed is unable to resize the home directory.

## Default root password and user when booting in a virtual machine

If you boot ParticleOS in a virtual machine using `mkosi vm`, the root password
is automatically set to `particleos` and a default user `particleos` with password
`particleos` is created as well.
