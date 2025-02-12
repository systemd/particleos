# ParticleOS

ParticleOS is a fully customizable immutable distribution implementing the
concepts described in
[Fitting Everything Together](https://0pointer.net/blog/fitting-everything-together.html).

The crucial difference that makes ParticleOS unique compared to other immutable
distributions is that users build the ParticleOS image themselves and sign it
with their own keys instead of installing vendor signed images. This allows
configuring the image to your liking by having full control over which
distribution is used as the base and which packages are installed into the
image.

The ParticleOS image is built using [mkosi](https://github.com/systemd/mkosi).
To build the image, run `mkosi -d <distribution> -f` from the ParticleOS
repository. Currently both `arch` and `fedora` are supported distributions.
Implementing support for a new distribution (that's already supported in mkosi)
is as simple as writing the necessary config files to install the required
packages for that distribution.

To update the system after installation, you clone the ParticleOS repository
or your fork of it and run `mkosi -ff sysupdate update --reboot` which will
update the system using `systemd-sysupdate` and then reboot.

## Building systemd from source

Sometimes ParticleOS adopts systemd features as soon as they get merged into
systemd without waiting for an official release. As a result it's recommended to
build systemd from source when building ParticleOS to make sure all required
features are supported:

```sh
git clone https://github.com/systemd/systemd
cd systemd
mkosi -f sandbox -- meson setup build
mkosi -f sandbox -- meson compile -C build
mkosi -t none
```

Then write the following to `mkosi.local.conf` in the ParticleOS repository to
use the artifacts from the systemd repository built by mkosi in ParticleOS:

```conf
[Content]
PackageDirectories=../systemd/build/mkosi.builddir/<distribution>~<release>~<arch>

[Runtime]
ExtraSearchPaths=../systemd/build
```

Make sure the distribution and release in `mkosi.local.conf` are identical in the
systemd checkout and the particleos checkout.

To build a newer systemd, run `git pull` in the systemd repository followed by
 `mkosi -f sandbox meson compile -C build` and `mkosi -t none`.

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
`systemd-repart --dry-run=no --empty=force /dev/<drive>` to install ParticleOS
to the system's drive. Finally, reboot into the target drive (not the USB) and
the regular profile (not the installer one) to complete the installation.

## Configuring systemd-homed after installation

After installing ParticleOS and logging into your systemd-homed managed user,
run the following to configure systemd-homed for the best experience:

```sh
homectl update \
    --auto-resize-mode=off \
    --disk-size=max \
    --luks-discard=on \
    --extra-luks-mount-options "user_subvol_rm_allowed,compress=zstd:1"
```

Disabling the auto resize mode avoids slow system boot and shutdown. Enabling
LUKS discard makes sure the home directory doesn't become inaccessible because
systemd-homed is unable to resize the home directory. The extra LUKS mount
options are BTRFS mount options to make image builds with `mkosi` faster by
compressing data on disk and allowing users to delete subvolumes.

## Default root password and user when booting in a virtual machine

If you boot ParticleOS in a virtual machine using `mkosi vm`, the root password
is automatically set to `particleos` and a default user `particleos` with password
`particleos` is created as well.
