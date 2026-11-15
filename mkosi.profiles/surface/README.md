# Surface Profile

Builds ParticleOS for Microsoft Surface Pro and Surface Laptop devices on
standard x86-64 UEFI firmware (no custom bootloader required).

## What it adds

| Layer | Details |
|---|---|
| Kernel | Replaces the stock distro kernel with the linux-surface patched kernel (touch / Type Cover / pen / camera fixes) |
| Touch daemon | iptsd - processes IPTS touch/pen frames from the firmware |
| Firmware | SOF audio firmware, Intel WiFi firmware, Surface-specific libwacom database |
| Power | Deep sleep preference, lid-state fix, PSR disabled (prevents flicker) |
| Type Cover | udev wakeup rule so suspend/resume works with the keyboard |

## Build

```sh
# Fedora base (recommended - best Surface kernel COPR coverage)
mkosi --profile obs-repos --profile desktop --profile gnome --profile surface build

# Debian base
mkosi --distribution debian --profile desktop --profile gnome --profile surface build
```

## Installing on a Surface

1. Boot the generated disk image from USB (or write it directly to the NVMe).
2. Secure Boot: either enroll the ParticleOS certificate in the Surface UEFI,
   or disable Secure Boot temporarily in **Surface UEFI -> Security -> Secure Boot**.
3. The image uses **systemd-boot** as the EFI bootloader - it registers itself
   with the UEFI boot manager automatically; no manual EFI entry needed.
4. First boot runs `preset-global` to enable all default services.

## Supported models

Tested targets (linux-surface patch coverage):

- Surface Pro 7, 7+, 8, 9, 10
- Surface Laptop 3, 4, 5, 6
- Surface Go 2, 3
- Surface Pro X (ARM - not covered by this x86 profile)

## Known limitations

- **Camera**: MIPI cameras on Pro 9/10 and Laptop 5/6 are not fully supported
  in mainline yet; frames may be unavailable until IPU6 support lands.
- **Secure Boot**: custom key enrollment into Surface UEFI is possible but not
  automated by this profile yet.
