![Arch Linux](https://archlinux.org/static/logos/archlinux-logo-dark-90dpi.ebdee92a15b3.png)
## Arch Linux Installation Guide
#### The goal of this Arch Linux installation guide is to provide an easier to interpret, while still chomprehensive how-to for installing Arch Linux on x86_64 architecture devices. This guide  assumes you are technically inclined and have a basic understanding of Linux. This installation guide was made with intentions of sticking to systemd (systemd-boot, systemd-networkd). Just because of the sheer amount of options and prefrences for networking tools & boot loaders, instructions for NetworkManager & GRUB were included to their respective sections as well. 
###### This guide is a mix of knowledge and information from the [ArchWiki](https://wiki.archlinux.org/title/Installation_guide)
----
### Contents
* [Live Media Creation](#live-media-creation)
* [Verify Boot Mode](#verify-boot-mode)
* [Inital Network Setup](#initial-network-setup)
    * [Ethernet](#ethernet)
    * [Wi-Fi](#wi-fi)
* [System Clock](#system-clock)
* [Disk Partitioning](#disk-partitioning)
* [Swap Space](#swap-space)
    * [Swap Partition](#swap-partition)
    * [Swapfile](#swapfile)
* [Format Partitions](#format-partitions)
* [Mount Partitions](#mount-partitions)
* [Install Essential Packages](#install-essential-packages)
* [Fstab](#fstab)
* [Change Root](#change-root)
* [Time Zone](#time-zone)
* [Localization](#localization)
* [Network Configuration](#network-configuration)
    * [Systemd-networkd](#systemd-networkd)
    * [NetworkManager](#networkmanager)
* [Initramfs](#initramfs)
* [Users & Passwords](#users-and-passwords)
* [Boot Loader](#boot-loader)
    * [Systemd-boot](#systemd-boot)
    * [GRUB](#grub)
* [Arch Linux Installation Complete](#arch-linux-installation-complete)

---
### Live Media Creation
if a system with access to linux commands is available, use [dd](https://wiki.archlinux.org/title/Dd) to create a bootable arch linux installation image on a usb/sd card from the downloaded [iso](https://archlinux.org/download/). there are several options -- for more, see how to  create an arch linux installer image, [here](https://wiki.archlinux.org/title/USB_flash_installation_medium)

the 'of' path should be replaced with the destination to the usb/sd card; insert the device and use [lsblk](https://wiki.archlinux.org/title/Lsblk#lsblk) to check path. it should be something like 'sdb'

>  **#**&ensp; `dd bs=4M if=/<path>/<to>/<archlinux>.iso of=/dev/<sdx> status=progress && sync`

when a bootable image has sucessfully been created from the iso, attach the device and boot into the live enviornment. 

secure boot must be disabled from the system [bios](https://en.m.wikipedia.org/wiki/BIOS) menu to boot the installation medium

----

### Verify Boot Mode
list the efivars directory

> **#**&ensp; `ls /sys/firmware/efi/efivars`

if the command returns the directory without error, then the system is booted in [uefi](https://wiki.archlinux.org/title/UEFI). if the directory doesn't exist the system may be booted in [bios](https://wiki.archlinux.org/title/Arch_boot_process#Under_BIOS) or [csm](https://en.wikipedia.org/wiki/Compatibility_Support_Module) mode

----

### Initial Network Setup
check if the [network interface](https://wiki.archlinux.org/title/Network_interface#Network_interfaces) is enabled with [iplink](https://man.archlinux.org/man/ip-link.8)

> **#**&ensp; `ip link`

if disabled, check the device driver -- see [ethernet#device-driver](https://wiki.archlinux.org/title/Network_configuration/Ethernet#Device_driver) or [wireless#device-driver](https://wiki.archlinux.org/title/Network_configuration/Wireless#Device_driver)
#### _Ethernet_
plug in an ethernet cable
#### _Wi-Fi_
make sure the card isn't blocked with [rfkill](https://wiki.archlinux.org/title/Network_configuration/Wireless#Rfkill_caveat)

> **#**&ensp; `rfkill list`

if the card is soft-blocked by the kernel, use this command:

> **#**&ensp; `rfkill unblock wifi`

the card could be hard-blocked by a hardware button or switch, e.g. a laptop. make sure this is enabled.
 
authenticate to a wireless network in an [iwd](https://wiki.archlinux.org/title/Iwctl) interactive prompt

> **#**&ensp; `iwctl`

list all wireless devices

> [iwd]**#**&ensp; `device list`

scan for networks

> [iwd]**#**&ensp; `station <device> scan`

list scanned networks

> [iwd]**#**&ensp; `station <device> get-networks`

finally, connect to specified network

> [iwd]**#**&ensp; `station <device> connect <SSID>`

verify connection

> [iwd]**#**&ensp; `station <device> show`

exit prompt using 'ctrl+c'

_° for wwan (mobile broadband) -- see [nmcli](https://wiki.archlinux.org/title/Mobile_broadband_modem#ModemManager)_

----

### System Clock
set system clock [timedatectl](https://man.archlinux.org/man/timedatectl.1)

> **#**&ensp; `timedatectl set-ntp true`

check status

> **#**&ensp; `timedatectl status`

----

### Disk Partitioning
list disk and block devices 

> **#**&ensp; `lsblk`
 
using the most desirable [partitioning](https://wiki.archlinux.org/title/Partition) tool([gdisk](https://wiki.archlinux.org/title/Gdisk), [fdisk](https://wiki.archlinux.org/title/Fdisk), [parted](https://wiki.archlinux.org/title/parted), etc) for your system, create a new gpt or mbr partition table, if one does not exist. a gpt table is required in uefi mode; an mbr table is required if the system is booted in legacy bios mode but a 
 
 if booted in uefi, create an  [efi](https://wiki.archlinux.org/title/EFI_system_partition)(512MB) [system partition](https://wiki.archlinux.org/title/EFI_system_partition); not necessary for legacy bios systems. create the required [root](https://wiki.archlinux.org/title/Root_directory#/)(10GB+) [partition](https://wiki.archlinux.org/title/Root_directory).

if the system has an existing efi partition, don't create a new one. do not format it or all data on the partition will be lost, rendering other operating systems potentially un-bootable. skip the 'mkfs.vfat' command in the [format partitions](#format-partitions) section below and mount the already existing efi partition if available

to create any stacked block devices for [lvm](https://wiki.archlinux.org/title/LVM), [system encryption](https://wiki.archlinux.org/title/Dm-crypt) or [raid](https://wiki.archlinux.org/title/RAID), do it now

----

### _Swap Space_
in order to create a [swap](https://wiki.archlinux.org/title/Partitioning_tools#Swap) consider creating either a [swap partition](https://wiki.archlinux.org/title/Swap#Swap_partition) or a [swapfile](https://wiki.archlinux.org/title/Swap#Swap_file) now. to share the [swap space](https://wiki.archlinux.org/title/Swap_file#Swap_space) system-wide with other operating systems or enable hibernation; create a linux swap partition. in comparison, a swapfile can change size on-the-fly and is more easily removed, which may be more desirable for a modestly-sized ssd

----

#### _Swap Partition_
if a swap partition was made, format it by replacing 'swap_partition' with it's  assigned block device path, e.g. sda2

> **#**&ensp; `mkswap /dev/<swap_partition>`

then activate it

> **#**&ensp; `swapon /dev/<swap_partition>`

----

#### _Swapfile_
to create a swapfile instead, use dd. the following command will create a 4gb swapfile 

> **#**&ensp; `dd if=/dev/zero of=/swapfile bs=1M count=<4096> status=progress`

format it

> **#**&ensp; `mkswap /swapfile`

then activate it

> **#**&ensp; `swapon /swapfile`

a swapfile without the correct permissions is a big security risk. set the file permissions to 600 with [chmod](https://wiki.archlinux.org/title/File_permissions_and_attributes#Changing_permissions)

> **#**&ensp; `chmod 600 /swapfile`

----

### Format Partitions
format the root partition just created with preferred [filesystem](https://wiki.archlinux.org/title/File_systems) and replace 'root_partition' with it's assigned [block device](https://en.m.wikipedia.org/wiki/Device_file#Block_devices) path, e.g. sda3

> **#**&ensp; `mkfs.<ext4> /dev/<root_partition>`

for uefi systems, if an efi partition was created, format it by replacing 'efi_partition' with it's assigned block device path, e.g. sda1

> **#**&ensp; `mkfs.vfat -F32 /dev/<efi_partition>`

----

### Mount Partitions
[mount](https://wiki.archlinux.org/title/Mount) root partition to /mnt

> **#**&ensp; `mount /dev/<root_partition> /mnt`

if created, mount the efi system [partition](https://wiki.archlinux.org/title/Partitioning#Example_layouts) to [/boot](https://wiki.archlinux.org/title/Partitioning#/boot)

> **#**&ensp; `mount /dev/<efi_partition> /mnt/boot`

----

### Install Essential Packages
[pacstrap](https://man.archlinux.org/man/pacstrap.8) base, [kernel](https://wiki.archlinux.org/title/Kernel) choice and if the system has an amd or intel cpu, install the coinciding [microcode](https://wiki.archlinux.org/title/Microcode) updates

> **#**&ensp; `pacstrap /mnt base linux linux-firmware nano sudo <cpu_manufacturer>-ucode`

----

### Fstab
generate an [fstab](https://wiki.archlinux.org/title/Fstab) file from detected mounted block devices, defined by uuid's

> **#**&ensp; `genfstab -U /mnt >> /mnt/etc/fstab`

check generated fstab

> **#**&ensp; `cat /mnt/etc/fstab`

----

### Change Root
[chroot](https://wiki.archlinux.org/title/Change_root) into freshly installed system

> **#**&ensp; `arch-chroot /mnt`

----

### Time Zone
set [time zone](https://wiki.archlinux.org/title/Time_zone)

> **#**&ensp; `ln -sf /usr/share/zoneinfo/<region>/<city> /etc/localtime`

generate /etc/adjtime with [hwclock](https://man.archlinux.org/man/hwclock.8)

> **#**&ensp; `hwclock --systohc`

_° this assumes the hardware clock is set to [utc](https://en.m.wikipedia.org/wiki/UTC). for more, see [system time#time standard](https://wiki.archlinux.org/title/System_time#Time_standard)_

----

### Localization
[edit](https://wiki.archlinux.org/title/Textedit) /etc/locale.gen and un-[comment](https://linuxhint.com/bash_comments/) 'en_US.UTF-8 UTF-8' or any other necessary [locales](https://wiki.archlinux.org/title/Locale) by removing the '#'

> **#**&ensp; `nano /etc/locale.gen`

use ctrl+x then 'y' to save and close [nano](https://wiki.archlinux.org/title/Nano)

_° for a different editor -- see [documents#editors](https://wiki.archlinux.org/title/List_of_applications/Documents#Text_editors)_

generate the locales

> **#**&ensp; `locale-gen`

[create](https://wiki.archlinux.org/title/Textedit) the [locale.conf](https://man.archlinux.org/man/locale.conf.5) file and set the [system locale](https://wiki.archlinux.org/title/Locale#Setting_the_system_locale) 

> **#**&ensp; `echo LANG=en_US.UTF-8 >> /etc/locale.conf`

----

### Network Configuration
create [hostname](https://wiki.archlinux.org/title/Hostname) file

> **#**&ensp; `echo <hostname> >> /etc/hostname`

add matching entries to [hosts](https://man.archlinux.org/man/hosts.5)

> **#**&ensp; `nano /etc/hosts`
```diff
127.0.0.1    localhost  
::1          localhost  
127.0.1.1    <hostname>.localdomain <hostname>
```

_° if the system has a permanently assigned ip address, use it instead of '127.0.1.1'_

install any desired [network managment](https://wiki.archlinux.org/title/Network_configuration) software. for this guide [systemd-networkd](https://wiki.archlinux.org/title/Systemd-networkd) is used by example. for an easier, configureless setup -- check out networkmanager just below

----

#### _Systemd-networkd_ 
install wpa_supplicant

> **#**&ensp;`pacman -S wpa_supplicant`

connect to the network with [wpa_passphrase](https://wiki.archlinux.org/title/Wpa_supplicant#Connecting_with_wpa_passphrase)

> **#**&ensp; `wpa_passphrase <ssid> <password> > /etc/wpa_supplicant/wpa_supplicant-<interface>.conf`

[enable & start](https://wiki.archlinux.org/title/Start/enable) the wpa_supplicant daemon reading the config file just created

> **#**&ensp; `systemctl enable --now wpa_supplicant@<interface>`

setup systemd-networkd for wireless networking

> **#**&ensp; `nano /etc/systemd/network/25-wireless.network`
```diff
[Match]
Name=<interface>

[Network] 
DHCP=yes
```
enable and start systemd-network service daemon

> **#**&ensp; `systemctl enable --now systemd-networkd`

_° for a static connection -- see [#static](https://wiki.archlinux.org/title/Systemd-networkd#Wired_adapter_using_a_static_IP); for ethernet -- see [#wired adapter](https://wiki.archlinux.org/title/Systemd-networkd#Wired_adapter_using_DHCP)_

----

#### _NetworkManager_
install [networkmanager](https://wiki.archlinux.org/title/NetworkManager)

> **#**&ensp; `pacman -S NetworkManager`

enable and start networkmanager service daemon

> **#**&ensp; `systemctl enable --now NetworkManager`

----

### _Initramfs_
_° creating an initramfs image isn't necessary since [mkinitcpio](https://wiki.archlinux.org/title/Mkinitcpio) was ran when pacstrap installed the kernel_

for [lvm](https://wiki.archlinux.org/title/Install_Arch_Linux_on_LVM#Adding_mkinitcpio_hooks), [system encryption](https://wiki.archlinux.org/title/Dm-crypt) or [raid](https://wiki.archlinux.org/title/RAID#Configure_mkinitcpio) modify [mkinitcpio.conf](https://man.archlinux.org/man/mkinitcpio.conf.5) then recreate the initramfs image with:

> **#**&ensp; `mkinitcpio -P`

----

### Users and Passwords
create a new user

> **#**&ensp; `useradd -m <username>`

add created user to the wheel group

> **#**&ensp; `usermod -aG wheel <username>`

uncomment '%wheel', by removing the '#'

> **#**&ensp; `EDITOR=nano visudo`

set created user [password](https://wiki.archlinux.org/title/Password)

> **#**&ensp; `passwd <username>`

set [root user](https://wiki.archlinux.org/title/Root_user) password

> **#**&ensp; `passwd`

disable login to [superuser/root](https://en.m.wikipedia.org/wiki/Root_user), locking password entry for root user. this will give the system increased security and a user can still be elevated within the wheel group to superuser priveleges with [sudo](https://wiki.archlinux.org/title/Sudo) and [su](https://wiki.archlinux.org/title/Su) commands

> **#**&ensp; `passwd -l root`

----

### Boot Loader 
install a linux-capable [boot loader](https://wiki.archlinux.org/title/Boot_loader). for simplicity and ease-of-use I recommend systemd-boot, not grub for uefi. systemd-boot will boot any configured efi image including windows operating systems. systemd-boot is not compatible with systems booted in legacy bios mode

----

#### _Systemd-boot_
install [systemd-boot](https://wiki.archlinux.org/title/Systemd-boot) in the efi system partition

> **#**&ensp; `bootctl --path=/boot install`

add a [boot entry](https://wiki.archlinux.org/title/Systemd-boot#Adding_loaders) and load installed microcode updates

> **#**&ensp; `nano /boot/loader/entries/<entry>.conf`
```diff
title <Arch Linux>  
linux /vmlinuz-linux  
initrd /<cpu_manufacturer>-ucode.img  
initrd /initramfs-linux.img  
options root=/dev/<root_partition> rw quiet log-level=0
```

_° if a different kernel was installed such as linux-zen, you would add '-zen' above to 'vmlinuz-linux' and 'initframs-linux'. this will boot the system using the selected kernel. for more -- see [kernel paramters](https://wiki.archlinux.org/title/Kernel_parameters#systemd-boot)_ 

edit [loader config](https://man.archlinux.org/man/loader.conf.5)

> **#**&ensp; `nano /boot/loader/loader.conf`
```diff
default <entry>.conf  
timeout <3>  
console-mode <max>  
editor <no>
```

verify entry is bootable

> **#**&ensp; `bootctl list`

----

#### _GRUB_
install [grub](https://wiki.archlinux.org/title/GRUB). also, install  [os-prober](https://archlinux.org/packages/?name=os-prober) to automatically add boot entries for other operating systems

> **#**&ensp; `pacman -S grub efibootmgr os-prober`
#### _° UEFI_
for systems booted in uefi mode,
install [efibootmgr](https://wiki.archlinux.org/title/EFISTUB#efibootmgr)

> **#**&ensp; `pacman -S efibootmgr`

and install grub to the efi partition

> **#**&ensp; `grub-install --target=x86_64-efi --efi-directory=/boot/grub --bootloader-id=GRUB`
----
#### _° BIOS_
othwerwise, if booted in bios mode; where path is the entire disk, not just a partition or path to a directory
install grub to the disk

> **#**&ensp; `grub-install --target=i386-pc /dev/sdx`

----

generate the [grub config](https://wiki.archlinux.org/title/GRUB#Configuration) file. this will automatically detect the arch linux installation

> **#**&ensp; `grub-mkconfig -o /boot/grub/grub.cfg`

_° if a warning that os-prober will not be executed appears then, un-comment 'GRUB_DISABLE_OS_PROBER=false' by removing the '#'_

> **#**&ensp; `nano /etc/default/grub`

----

### Arch Linux Installation Complete
exit chroot using ctrl+d

unmount all partitions; check if busy

> **#**&ensp; `unmount -R /mnt`

reboot the system

> **#**&ensp; `reboot`
