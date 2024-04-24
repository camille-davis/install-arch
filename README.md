# Installing Arch Linux on MacBook Pro 8,3 (Late 2011)

* Linux only (no dual boot).
* Requires ethernet connection for initial install.

TODO: Reorder steps more logically.

## References

* [Arch Wiki - MacBook Pro 8,x](https://wiki.archlinux.org/title/MacBookPro8,x)
* [Arch Wiki - Laptop/Apple](https://wiki.archlinux.org/title/Laptop/Apple)
* [Arch Wiki - Installation Guide](https://wiki.archlinux.org/title/Installation_guide)
* [Philp @ Medium - Arch Linux Running on my MacBook](https://medium.com/@philpl/arch-linux-running-on-my-macbook-2ea525ebefe3)

## Pre-installation

Follow the pre-installation instructions from the [Installation Guide](https://wiki.archlinux.org/title/Installation_guide). On step 1.4 (booting the live environment) add some kernel parameters to prevent boot from freezing:

Hold the `Option` key while starting up. Select the Arch Linux installer and press `e` to edit. Add the following kernel parameters at the end of the `linux` line:

```
radeon.modeset=0 i915.modeset=0
```

## In the Live Environment

### Configure Filesystems

#### Encrypted Root Partition

Note: I had to encrypt my root partition for work requirements, but you don't necessarily have to.

Wipe the root partition with zeros for a better encryption:
```
cryptsetup open --type plain -d /dev/urandom /dev/sda2 to_be_wiped
dd if=/dev/zero of=/dev/mapper/to_be_wiped status=progress
cryptsetup close to_be_wiped
```

Encrypt the partition:
```
cryptsetup -y -v luksFormat /dev/sda2
```

Backup LUKS header onto another block device:
```
cryptsetup luksHeaderBackup /dev/sda2 --header-backup-file /luks_header_backup
```

Format, label, and mount the partition:
```
cryptsetup open /dev/sda2 root
mkfs.ext4 /dev/mapper/root -L "Arch Linux"
mount /dev/sda2 /mnt
```

#### EFI Boot Partition

The EFI boot partition is already formatted, so you just have to mount it:
```
mount --mkdir /dev/sda1 /mnt/boot
```

TODO: This partition comes with only 200M; instructions should be revisited to allow more space.

#### Generate fstab (Filesystems Table)

Generate an fstab file (this should create entries for sda2 and sda1; we will add a swapfile later).
```
genfstab -U /mnt >> /mnt/etc/fstab
```

Open `/mnt/etc/fstab` and make sure both entries are there.

### Install Essential Packages

Make sure you are connected to Ethernet, then run:
```
pacstrap -K /mnt base linux linux-firmware
```

## In the Chrooted Environment

Chroot into the new system:
```
arch-chroot /mnt
```

### Install More Packages

Get the cute pacman progress bar: open `/etc/pacman.conf` and uncomment `ILoveCandy`.

Install packages with `pacman -S [package name]`:
* vim (or editor of your choice)
* base-devel (needed to create packages)
* git
* sudo
* dhcpcd (ethernet won't work without it, which is strange because it worked in the pacstrap command. Maybe that command uses its own DHCP?)
* iwd (for wifi)

Make console font bigger and better:
```
pacman -S terminus-fonts
```

In `/etc/vconsole.conf` set `FONT=ter-125n`.

#### Downgrade xz

Downgrade xz to 5.4.6 or earlier due to xz exploit (better safe than sorry). Download it from the [Arch Archive - xz](https://archive.archlinux.org/packages/x/xz/) then run:
```
pacman -R xz
pacman -U /path/to/download
```

Delete any newer versions of xz from `/var/cache/pacman/pkg`.

Exclude it from sync: in `/etc/pacman.conf` uncomment `IgnorePkg` and add `xz`.

Update everything else:
```
pacman -Syu
```

### Do Some Config

Set hostname in `/etc/hostname`.

Set root password:
```
passwd
```

Set timezone:
```
ln -sf /usr/share/zoneinfo/[your region]/[your city] /etc/localtime
hwclock --systohc
```

Set locale:
* Edit `/etc/locale.gen` and uncomment `en_US.UTF-8 UTF-8` and other locales.
* Create `/etc/locale.conf` and set `LANG=en_US.UTF-8`

### Configure Swapfile for Hibernation

Create 20GB swapfile:

```
dd if=/dev/zero of=/swapfile bs=1G count=20 status=progress
chmod 0600 /swapfile
mkswap /swapfile
swapon /swapfile
```

Make it persistent by adding an entry to fstab:
```
/swapfile none swap defaults 0 0
```

Check for errors (don't reboot until fixed): `mount -a`

Set a low swappiness value permanently so we only use it for hibernation: create configuration file at `/etc/sysctl.d/99-swappiness.conf` and add `vm.swappiness = 1`.

TODO: Can I do this at the moment of partitioning the disk instead?

### Setup Boot Loader

Configure mkinitcpio at: `/etc/mkinitcpio.conf`:
* Add modules:
  * `usbhid xhci_hcd` (to use USB keyboard)
  * `ahci libahci` (to fix fs not found error - TODO: Can I remove this?)
* Add hooks:
  * `encrypt` (place somewhere after udev)
  * `keyboard` (somewhere before autodetect)

Add compression to save space on the boot partition: uncomment `compression xz` and add `options -9`.

Install systemd-boot (works well with native Apple boot loader):
```
bootctl --path=/boot install
```

Set default loader configuration in `boot/loader/loader.conf`:
```
default arch.conf
timeout 0
console-mode keep
```

Set specific loader configuration in `boot/loader/entries/arch.conf`:
```
title   Arch Linux
linux   /vmlinuz-linux
initrd  /initramfs-linux.img
options cryptdevice=/dev/sda2:root root=/dev/mapper/root resume=UUID=[swap device uuid] resume_offset=[swap file offset]
```

You can get the resume offset by running:
```
filefrag -v /swapfile
```

Look for physical offset column in first row, ends in `'..'`.

Finally, renegerate initramfs:
```
mkinitcpio -P
```

TODO: Setup and test hibernation.

## Let's Reboot!

Reboot the machine. It should now boot from our new boot loader.

### Create Non-Root User

Create a user (you can't run `makepkg` as root) in groups `wheel`, `users`.

### Setup Wifi

#### Setup Kernel Modules

I had issues with `b43` not connecting properly to certain routers, so we will use `wl.`

Install `broadcom-wl-dkms` and dependencies:
```
pacman -S linux_headers
pacman -S broadcom-wl-dkms
modprobe wl
```

Remove modules known to conflict with `wl`:
```
rmmod b43
rmmod bcma
```

Create `/etc/modprobe.d/10_wl.conf` and add:
```
blacklist bcma
blacklist b43
```

Check that kernel driver in use is `wl`:
```
lspci -v
```

Regenerate initramfs again:
```
sudo mkinicpio -P
```

TODO: Do I need to do this everytime I change modules?

#### Setup Wifi Services

Enable iwd:
```
systemctl enable iwd
systemctl start iwd
```

In `/etc/iwd/main.conf`, add the following config to enable autoconnect and built-in DHCP (using systemd-resolved):
```
[General]
EnableNetworkConfiguration=true
[Settings]
AutoConnect=true
```

Make sure systemd-resolved is enabled; if not, enable and start it.
```
systemctl enable systemd-resolved
systemctl start systemd-resolved
```

TODO: systemd-resolved sometimes stops working after suspend, figure out why. In the meantime just restart the service.

Disable dhcpcd or there will be a conflict:
```
systemctl disable dhcpcd
systemctl stop dhcpcd
```

TODO: ethernet currently needs dhcpcd; figure out a way to switch between wifi and ethernet without having to enable/disable services. Maybe we can configure ethernet to use systemd-resolved.

#### Connect to Wifi

Get networks and connect to wifi:
```
iwctl
device list
station wlan0 (or whatever the device is) scan
station wlan0 get-networks
station wlan0 connect (or connect-hidden) [ssid]
```

I also had to add this section in `/var/lib/iwd/[name of network].psk`:
```
[IPv4]
Address=[my local ip]
Netmask=255.255.255.0
Gateway=[gateway local ip]
Broadcast=[last ip of subnet]
```

## Power and Fans

(Copied from [Philp @ Medium - Arch Linux Running on my MacBook](https://medium.com/@philpl/arch-linux-running-on-my-macbook-2ea525ebefe3) but have not tested it.)

### Get mbpfan

Make sure you have `coretemp` and `applesmc` modules. To turn on a module:
```
modprobe [name of module]
```

Check module is active:
```
lsmod | grep [name of module]
```

Make a directory for AUR downloads, e.g. `~/builds`. In this directory, download mbpfan-git:
```
git clone https://aur.archlinux.org/mbpfan-git.git
cd mpbfan-git
makepkg
pacman -U mbpfan-git
systemctl enable mbpfan-git
systemctl start mbpfan-git
```

#### More Services

Install some more services:
* thermald
* cpupower

In `/etc/default/cpupower` set `governor` to `powersave`.

## Setup Desktop Environment

I had some issues with Wayland so let's use Xorg:
```
pacman -S xorg xorg-server
```

Install Gnome and its desktop manager:
```
pacman -S gnome
systemctl enable gdm
```

Install Intel display drivers:
```
pacman -S xf86-video-intel
```

## More Customization

Install gnome-macos-remap to switch `Cmd` and `Ctrl` keys.

Disable 'Activities' view:
```
gsettings set org.gnome.mutter overlay-key ''
```

Set power button to suspend, in `/etc/systemd/logind.conf` set:
```
HandlePowerKey=suspend
```

## TODO

* Setup hibernation.
* Figure out how to switch between wifi and ethernet without changing modules or services.
* Fix DHCP breaking after suspend.
* Make boot partition bigger.
* Figure out how to export Gnome keyboard shortcuts.
