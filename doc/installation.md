UEFI installation procedure
===========================

1. Pre-installation (from another computer)
-------------------------------------------

- [ ] Visit [the download page](https://www.archlinux.org/download/)
- [ ] Download the PGP signature `archlinux-<date>-<arch>.iso.sig`
- [ ] Download latest ISO `archlinux-<date>-<arch>.iso`
- [ ] Download associated checksums `sha1sums.txt` and `md5sums.txt`
- [ ] Validate download
    * `md5sum --check md5sums.txt`
    * `sha1sum --check sha1sums.txt`
- [ ] Verify PGP signature against ISO
    * `gpg --keyserver-options auto-key-retrieve --verify archlinux-<date>-<arch>.iso`
- [ ] Verify key fingerprint against [Archlinux developers page](https://www.archlinux.org/people/developers/)
- [ ] Create a bootable USB flash drive
    * `dd if=archlinux-<date>-<arch>.iso of=/dev/<usb-drive> bs=1M status=progress`
    * `sync`


[//]: # Find appropriate names for sections

- [ ] Insert the bootable USB flash drive into the computer
- [ ] Plug an ethernet cable into the computer
- [ ] Boot the computer
- [ ] Select "Boot Arch Linux" or similar
- [ ] Verify the ``efivars`` folder exists (UEFI boot mode)
    * `ls /sys/firmware/efi/efivars`
- [ ] Check network access
    * `ping archlinux.org`
- [ ] Enable NTP
    * `timedatectl set-ntp true`
[//]: # Define timezone?
- [ ] Check time was synchronized
    * `timedatectl status`
- [ ] Enter ``gdisk`` to create a GPT partition table
    * `gdisk /dev/sda`
- [ ] Create a new empty partition table
    * `o`
- [ ] Create the EFI system partition
    * `n`, create new partition
    * `<Enter>`, default partition number (should be `1`)
    * `<Enter>`, default first sector (should be `2048`)
    * `+512M`, 512MiB size
    * `ef00`, EFI system type
- [ ] Create the Linux filesystem partition
    * `n`, create new partition
    * `<Enter>`, default partition number (should be `2`)
    * `<Enter>`, default first sector (should be `1050623`)
    * `+2G`, 2GiB size
    * `8300`, Linux filesystem type
- [ ] Create the LVM partition
    * `n`, create new partition
    * `<Enter>`, default partition number (should be `3`)
    * `<Enter>`, default first sector (should be `5244928`)
    * `<Enter>`, use remaining space
    * `8e00`, Linux LVM type
- [ ] Write partition data and exit
    * `w`
- [ ] Create a FAT32 filesystem on the EFI partition
    * `mkfs.fat -F32 /dev/sda1`
- [ ] Create an encrypted boot container on the Linux filesystem partition (choose a good passphrase)
    * `cryptsetup luksFormat --type luks2 --cipher aes-xts-plain64 --key-size 512 /dev/sda2`
- [ ] Make the boot container available for formatting
    * `cryptsetup open /dev/sda2 cryptboot`
- [ ] Create an EXT2 filesystem on the system partition
    * `mkfs.ext2 /dev/mapper/cryptboot`
- [ ] Create an encrypted container for root and swap on the Linux LVM partition (choose a good passphrase)
    * `cryptsetup luksFormat --type luks2 --cipher aes-xts-plain64 --key-size 512 /dev/sda3`
- [ ] Make the container available for formatting
    * `cryptsetup open /dev/sda3 cryptlvm`
- [ ] Initialize physical volume
    * `pvcreate /dev/mapper/cryptlvm`
- [ ] Initialize volume group
    * `vgcreate vg0 /dev/mapper/cryptlvm`
- [ ] Create swap logical volume
    * `lvcreate --size <ram-size>G vg0 --name swap`
- [ ] Create root logical volume
[//]: # Create system partion and data partition?
    * `lvcreate --extents 100%FREE vg0 --name root`
- [ ] Format swap volume
    * `mkswap /dev/mapper/vg0-swap`
- [ ] Activate swap volume
    * `swapon /dev/mapper/vg0-swap`
- [ ] Format root volume
    * `mkfs.btrfs /dev/mapper/vg0-root`
- [ ] Mount partitions on the live system
    * `mount /dev/mapper/vg0-root /mnt`
    * `mkdir /mnt/boot`
    * `mount /dev/mapper/cryptboot /mnt/boot`
    * `mkdir /mnt/boot/efi`
    * `mount /dev/sda1 /mnt/boot/efi`
- [ ] Install base system and useful packages
    * `pacstrap /mnt base base-devel linux linux-firmware lvm2 btrfs-progs grub-efi-x86_64 vim git efibootmgr`
- [ ] Generate `fstab` without pseudofs mounts and using UUID for source identifiers
    * `genfstab -pU /mnt >> /mnt/etc/fstab`
- [ ] Chroot into the system
    * `arch-chroot /mnt`
- [ ] Configure timezone
    * `ln --symbolic --force /usr/share/zoneinfo/Europe/Paris /etc/localtime`
- [ ] Configure hardware clock as UTC
    * `hwclock --systohc --utc`
- [ ] Choose locales to use
    * `echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen`
    * `echo "fr_FR.UTF-8 UTF-8" >> /etc/locale.gen`
- [ ] Generate locales
    * `locale-gen`
- [ ] Define system language
    * `echo "LANG=en_US.UTF-8" >> /etc/locale.conf`
- [ ] Define a hostname
    * `echo "goodhostname" > /etc/hostname`
- [ ] Set a strong root password
    * `passwd`
- [ ] Enable lvm2, btrfs and encryption in mkinitcpio
    * `sed -i "s/MODULES=.*/MODULES=(btrfs)/g" /etc/mkinitcpio.conf`
    * `sed -i "s/HOOKS=.*/HOOKS=(base udev autodetect modconf block keymap encrypt lvm2 resume filesystems keyboard fsck shutdown)/g" /etc/mkinitcpio.conf`
- [ ] Get access to btrfs early on in the boot process, useful to repair fileystem
    * `sed -i 's/BINARIES=.*/BINARIES=("/usr/bin/btrfs")/g' /etc/mkinitcpio.conf`
- [ ] Change GRUB configuration to work with encrypted devices
    * `echo "GRUB_ENABLE_CRYPTODISK=y" >> /etc/default/grub`
    * `sed -i 's#^GRUB_CMDLINE_LINUX=.*#GRUB_CMDLINE_LINUX="cryptdevice=UUID=$(blkid /dev/sda3 -s UUID -o value):lvm resume=/dev/mapper/vg0-swap quiet splash"#g' /etc/default/grub`
- [ ] Create target output folder for GRUB configuration
    * `mkdir /boot/grub`
- [ ] Apply configuration
    * `grub-mkconfig --output /boot/grub/grub.cfg`
- [ ] Install GRUB to disk
    * `grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ArchLinux`
[//]: # Missing instructions so that boot auto decrypts:
[//]: # $ dd bs=512 count=8 if=/dev/urandom of=/etc/key
[//]: # $ chmod 400 /etc/key
[//]: # $ cryptsetup luksAddKey /dev/sda2 /etc/key
[//]: # $ echo "cryptboot /dev/sda2 /etc/key luks" >> /etc/crypttab
[//]: # Missing instructions so that LVM auto decrypts:
[//]: # $ dd bs=512 count=8 if=/dev/urandom of=/keyfile.bin
[//]: # $ chmod 000 /keyfile.bin
[//]: # $ cryptsetup luksAddKey /dev/sda3 /keyfile.bin
[//]: # $ sed -i 's\^FILES=.*\FILES="/keyfile.bin"\g' /etc/mkinitcpio.conf
[//]: # $ mkinitcpio -p linux
[//]: # $ chmod 600 /boot/initramfs-linux*
- [ ] Create a new user
    * `useradd --create-home --gid users --groups wheel $USERNAME`
- [ ] Set a password for the user
    * `passwd $USERNAME`
- [ ] Allow users from the `wheel` group to use sudo
    * `sed -i 's/# %wheel ALL=(ALL) ALL/%wheel ALL=(ALL) ALL/g' /etc/sudoers`
- [ ] Cleanup and shutdown
    * `exit`
    * `umount --recursive /mnt`
    * `swapoff --all`
    * `shutdown now`
