- [ ] Download an [installation image](https://www.debian.org/distrib/), possible a `non-free` one containing firmwares required by the hardware (most likely needed if installatin from wifi)
- [ ] Write the ISO to a bootable drive
- [ ] Select "Install" from the main menu
- [ ] Follow steps until the partitioning step
- [ ] Select "Guided - use entire disk and set up encrypted LVM"
- [ ] Select the proper disk
- [ ] Select "Separate /home partition"
- [ ] Answer "Yes" to writing changes to disk
- [ ] Cancel overriding partition with random data, unless you have time for it
- [ ] Select a robust passphrase and write it down in KeePass
- [ ] Use 70% of the LVM volume group
[//]: # Reconsider this % choice?
- [ ] Check partitioning is correct (home, root and swap volumes, /boot primary) and select "Finish partitioning and write changes to disk"
- [ ] Answer "Yes" to writing changes to disk
- [ ] Answer "No" to scanning another CD or DVD
- [ ] Select a proper country for mirror
- [ ] Select "deb.debian.org" as archive mirror
- [ ] Only select "standard system utilities" on the "Software selection" step
- [ ] Answer "Yes" to installing GRUB
- [ ] Reboot into the system and login as "root"

[//]: # Encrypt boot partition
- [ ] Assert the LUKS format version on the root device
    * `cryptsetup luksDump /dev/sdaX`
- If the output contains `Version: 2`, then:
    * Reboot the computer
    * Enter `e` at the GRUB menu and add `break=mount` to the end of the `linux` line
    * Press F-10 to boot
    * `cryptsetup luksConvertKey --pbkdf pbkdf2 /dev/sdaX`
    * `cryptsetup convert --type luks1 /dev/sdaX`
    * Assert the LUKS format contains `Version 1` now: `cryptsetup luksDump /dev/sdaX`
    * CTRL-ALT-DELETE to reboot
- [ ] Boot into the system as "root"
- [ ] Move boot to root
    * `mount -o remount,ro /boot`, so that it's not modified while being copied
    * `cp -axT /boot /boot.tmp`
    * `umount /boot`
    * `rmdir /boot`
    * `mv -T /boot.tmp /boot`
- [ ] Comment out the `/boot` entry from `/etc/fstab`
- [ ] Add the `CRYPTODISK` module to GRUB
    * `echo "GRUB_ENABLE_CRYPTODISK=y" >> /etc/default/grub`
- [ ] Update and reinstall GRUB
    * `update-grub`
    * `grub-install /dev/sda`
- [ ] Verify that grub.cfg has entries for insmod cryptodisk and insmod luks
    * `grep 'cryptodisk\|luks' /boot/grub/grub.cfg`
- If not:
    * Add `cryptodisk` and `luks` to `GRUB_PRELOAD_MODULES` in `/etc/default/grub` and re-install grub
- [ ] Reduce the number of iterations of the PBKDF algorithm, since GRUB runs under tighter constraints and it would take too long otherwise
    * `cryptsetup luksChangeKey --pbkdf-force-iterations 500000 /dev/sda5`
- [ ] Reboot to test GRUB decrypting, there should be a "Enter passphrase" prompt
- [ ] Log back in as "root"

[//]: Prevent having to type the passphrase twice
- [ ] Generate an encryption keyfile and place it in a LUKS key slot
    * `mkdir -m 0700 /etc/keys`
    * `umask 0077 && dd if=/dev/urandom bs=1 count=64 of=/etc/keys/root.key conv=excl,fsync`
    * `cryptsetup luksAddKey --key-slot 1 /dev/sda5 /etc/keys/root.key`
      * /!\ Make sure the new key is inserted in the slot *after* the currently enabled one. This is because GRUB will try to open keys in order, so we want the first enabled key to be the one that GRUB wants, not the keyfile.
    * `cryptsetup luksDump /dev/sda5`, you should see "Key Slot 1: ENABLED"
- Inside `/etc/crypttab`:
    * [ ] Replace `none` with `/etc/keys/root.key`
    * [ ] Append `,key-slot=1` to `luks,discard`
- [ ] Inside `/etc/cryptsetup-initramfs/conf-hook`, uncomment and edit `KEYFILE_PATTERN` with value `"/etc/keys/*.key"` 
- [ ] Set `UMASK` to root-only access to avoid leaking key material
    * `echo UMASK=0077 >> /etc/initramfs-tools/initramfs.conf`
- [ ] Re-generate the initramfs image, and verify that it has the restrictive permissions and includes the key
    * `update-initramfs -u -k all`
    * `stat -L -c "%A  %n" /initrd.img`
    * `lsinitramfs /initrd.img | grep "^cryptroot/keyfiles/"`
- [ ] Add a network entry if using Wi-Fi, creating a new file under `/etc/network/interfaces.d/` containing:
    > iface <iface> inet dhcp
    > 
    > wpa-ssid SSID
    > 
    > wpa-psk PASSPHRASE
    > 
    > dns-nameservers 8.8.8.8 8.8.4.4
- [ ] Activate the interface
    * `ifup <iface>`
- [ ] Update and upgrade packages
    * `apt update`
    * `apt full-upgrade`
   
[//]: Configure ghub and clone system-setup
- [ ] Install some required packages
    * `apt install gnupg2 git software-properties-common`
- [ ] Follow steps to install GitHub CLI
    * https://github.com/cli/cli/blob/trunk/docs/install_linux.md#debian-ubuntu-linux-apt
- [ ] Generate a new SSH key
    * `ssh-keygen`
- [ ] Authenticate into GitHub as your user
    * `gh auth login`
    * Choose `SSH` as a preferred protocol, your SSH key will be uploaded as part of the authentication process
    * Choose to authenticate using your GitHub credentials
    * Choose "Login with a web browser"
    * Visit https://github.com/login/device/ to enter the token
- [ ] Clone `system-setup`
    * `gh repo clone lburg/system-setup`
    
[//]: Run system-setup
- [ ] Install Ansible
    * `apt install ansible`
- [ ] Install `sudo` and add your user to the group
    * `apt install sudo`
    * `adduser lburg sudo`
- [ ] Run the `site.yml` playbook
    * First make sure to delete interfaces manually configures in `/etc/network/interfaces{,.d}`, we will install a network manager
    * `ansible-playbook site.yml`

- [ ] Create SSH keys (logged in as the user)
    * `mkdir .ssh`
    * `cd .ssh`
    * `ssh-keygen`

Sources :

- https://www.dwarmstrong.org/fde-debian/
- https://cryptsetup-team.pages.debian.net/cryptsetup/encrypted-boot.html
- https://operandimodus.wordpress.com/2015/11/05/how-to-decrypt-and-mount-a-luks-encrypted-volume/







