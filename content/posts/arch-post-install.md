+++
title = 'Arch Post Install'
date = 2025-08-23T07:18:14Z
draft = false
+++

I decided to buy a new laptop so is become the time perform a new installation of Arch, in the past I already installed it from scratch, trying to fine tune every single settings, was really funny but this time I will opt for a more chilly approach and I will use [Archinstall](https://archinstall.archlinux.page/).

Because I would like to use different technologies I will test my installation first in [VirtualBox](https://www.virtualbox.org/) leveraging the installation over SSH as I explained on a past article [Install Arch with SSH](posts/install-arch-with-ssh/)

## Overwiew

I will install Arch Linux with the following settings/technologies:

- TPM 2.0
- Secure Boot
- BTRFS with snapthots on LVM
- UKI (Unified Kernel Images)
- Swap with Zram

## Installation

It is not possible to install Arch with Secure Boot enabled so before start the installation ensure is disabled on your UEFI. The awasome feature of ArchInstall is that you can install Arch declaratively by importing your settings from a `.json` file, since I already done my favorite installation file I just need to import it and run the installer.

One the installation is done select the last option **reboot system**.

## Post-Installation

### mkinitcpio

mkinitcpio is used to build the initramfs (initial RAM filesystem), which is required during the early stages of the boot process. This contains the necessary drivers, hooks and tools to unlock disks, detect hardware, mount the root filesystem and hand over control to the real system. As we are using TPM, UEFI and UKI, is required to adjust the hooks settings in the file `/etc/mkinitcpio.conf.d/hooks.conf`.

```config
HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt filesystems fsck)
```

- `base` → essential binaries/libs
- `systemd` → systemd-based initramfs
- `autodetect` → only include drivers for your hardware
- `microcode` → CPU microcode updates
- `modconf` → load extra kernel modules
- `kms` → early KMS for smooth graphics
- `sd-vconsole` → keyboard/layout/font early at boot
- `sd-encrypt` → unlock LUKS (supports TPM2, Secure Boot, UKI)
- `block` → basic block device support
- `filesystems` → mount root fs
- `fsck` → run fsck if needed

This configuration moves us from the default udev-based hook stack (`udev`, `encrypt`, …) to the systemd-based hooks (`sd-*`). Additionally, the kernel command line must be adjusted by editing the file `/etc/kernel/cmdline`, to ensure uses [systemd kernel parameters](https://wiki.archlinux.org/title/Dm-crypt/System_configuration#Using_systemd-cryptsetup-generator):

```config
rd.luks.name=<UUID>=<name> ...  rd.luks.options=<UUID>=password-echo=masked,no-read-workqueue,no-write-workqueue,discard
```

Here's what that kernel-cmdline syntax does:

- `rd.luks.name=<UUID>=cryptlvm` — unlocks the LUKS container with the specified UUID and creates the mapper device `/dev/mapper/cryptlvm`.
- `root=/dev/mapper/cryptroot` — points the kernel to your root filesystem directly.
- `rd.luks.options=<UUID>=` — pass dm-crypt options for that specific LUKS device at early boot:
  - `password-echo=masked` — show masked feedback (●●●) when typing the passphrase in initramfs.
  - `no-read-workqueue,no-write-workqueue` — disable dm-crypt workqueues (can reduce latency on some SSD/NVMe; see [Arch Wiki: Disable workqueue](https://wiki.archlinux.org/title/Dm-crypt/Specialties#Disable_workqueue_for_increased_solid_state_drive_(SSD)_performance)).
  - `discard` — allow TRIM to pass through the encrypted device (improves long-term SSD performance; see [Arch Wiki: Discard/TRIM](https://wiki.archlinux.org/title/Dm-crypt/Specialties#Discard/TRIM_support_for_solid_state_drives_(SSD))). Before enabling, verify your disk [supports TRIM](https://wiki.archlinux.org/title/Solid_state_drive#TRIM).
- `rootflags=subvol=@ rootfstype=btrfs` — mount options for your Btrfs root subvolume.

### Secure Boot

Before enabling **Secure Boot** in your firmware, you must first set up the boot loader with:

```sh
sudo bootctl install
```

This copies the `systemd-boot` EFI binary into your [EFI System Partition](https://en.wikipedia.org/wiki/EFI_system_partition) and creates the loader configuration files. After installation, you may see warnings about insecure permissions on `/boot` and the random seed file. To fix them se the correct permissions:

```sh
sudo chmod 700 /boot
sudo chmod 600 /boot/loader/random-seed
```

Then reboot into your UEFI settings, leave Secure Boot off but configure it in **setup mode** (refer to your UEFI manufaturer's guide on how to do that). To confirm you are in **setup mode** run the command `sudo sbctl status` you should see `Setup Mode: ✗ Enabled`.

Now create and enroll your Secure Boot keys:

```sh
# Create the key
sudo sbctl create-keys

# Enroll the key
sudo sbctl enroll-keys --microsoft
```

> Omit `--microsoft` if you don't want to add Microsoft's keys.

Check the secure boot status again with `sbctl status` you should see `Setup Mode: ✗ Enabled`.
sbctl should now be installed, but Secure Boot will not work until the boot files have been signed using the keys you have just created. Therefore, check which files need to be signed in order for Secure Boot to work:

```sh
sbctl verify
```

The command will show you all the unsigned `.efi` files. for each one run the command:

```sh
sudo sbctl sign --save </path/to/efi/file>
```

The `--save` flag tells sbctl to create a Pacman hook that will automatically re-sign the specified EFI file whenever the Linux kernel, systemd, or the bootloader is updated.
Last point is to enter again UEFI and enable Secure Boot.

### TPM 2.0

Enrolling a key in the TPM allows your system to unlock an encrypted disk automatically, but only if the machine is in a trusted state. Instead of typing a long passphrase every time you boot, the TPM can safely release the secret when the firmware and boot process look as expected. One common way to do this is to bind the key to PCR 7, which reflects the Secure Boot state. This makes sure the disk only unlocks when Secure Boot is active, blocking untrusted bootloaders. Secure Boot has to be in user mode, otherwise it doesn't really protect anything and an attacker could slip in their own bootloader. Because PCR 7 depends on Secure Boot's certificates, things like firmware updates or new keys can change its value. When that happens, the TPM won't unlock the disk automatically, so you'll need to fall back to your recovery passphrase.

The first step is to generate a recovery key that will save the day if something goes wrong in the future and you are no longer able to boot your system:

```sh
sudo systemd-cryptenroll /dev/nvme0n1p<N> --recovery-key
```

We will now enroll our system firmware and secure boot state. This will enable our TPM to unlock our encrypted drive as long as the state hasn't changed:

```sh
sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=0+7 --tpm2-with-pin=yes /dev/nvme0n1p<N>
```

> Using a PIN with TPM unlocking give you a good balance in terms of security. It prevents the disk from being unlocked completely unattended while avoiding the need to type in a long passphrase every time the computer is started up.

## Credits

I was inspired by this amazing guide that covers the entire manual installation process: [Arch Linux install with full disk encryption using LUKS2 - Logical Volumes with LVM2 - Secure Boot - TPM2 Setup](https://github.com/joelmathewthomas/archinstall-luks2-lvm2-secureboot-tpm2)
