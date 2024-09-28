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

There is no installer yet for ParticleOS. To install it, build the ParticleOS
image from another Linux system and `dd` it to the disk of the system you want
to run ParticleOS on.

# Configuring systemd-homed after installation

After installing ParticleOS and logging into your systemd-homed managed user,
run the following to configure systemd-homed for the best experience:

```sh
homectl update \
    --auto-resize-mode=off \
    --disk-size=max \
    --luks-discard=on \
    --extra-luks-mount-options "user_subvol_rm_allowed,compress=zstd:1"
```

Disabling the auto resize mode avoids slow system boot and shutdown. Disabling
LUKS discard makes sure the home directory doesn't become inaccessible because
systemd-homed is unable to resize the home directory. The extra LUKS mount
options are BTRFS mount options to make image builds with `mkosi` faster by
compressing data on disk and allowing users to delete subvolumes.
