![Arch Linux](https://archlinux.org/static/logos/archlinux-logo-dark-90dpi.ebdee92a15b3.png)
## Arch Linux Installation Guide
#### The goal of this Arch Linux installation guide is to provide an easier to interpret, while still chomprehensive how-to for installing Arch Linux on x86_64 architecture devices. This guide  assumes you are technically inclined and have a basic understanding of Linux. I made this guide with a purpose of sticking to systemd. Just because of the sheer amount of options for networking tools and boot loaders, I added NetworkManager to the networking configuration section and GRUB to the boot loader section. I highly prefer systemd-networkd and systemd-boot; when they work, well... they just always work, but they do have slightly more setup involved. For this reason I prioritized systemd based tools first.
###### This guide is a mix of knowledge and information taken directly from the [ArchWiki](https://wiki.archlinux.org/title/Installation_guide)
----
### <u>Live Media Creation</u>
if your system can access linux commands, use [dd](https://wiki.archlinux.org/title/Dd) to create a bootable installation image on a usb/sd card from the downloaded [iso](https://archlinux.org/download/). there are several options -- for more, see how to  create an installer image [here](https://wiki.archlinux.org/title/USB_flash_installation_medium)

the 'of' path should be replaced with the destination to your usb/sd; insert the device and use [lsblk](https://wiki.archlinux.org/title/Lsblk#lsblk) to check path. it should be something like 'sdb'

> $ `dd bs=4M if=/<path>/<to>/<archlinux>.iso of=/dev/<sdx> status=progress && sync`

when you've sucessfully created a bootable image from the iso, attach the device and boot into the live enviornment. secure boot must be disabled from the system [bios](https://en.m.wikipedia.org/wiki/BIOS)menu to boot the installation medium

----

### <u>Verify Boot Mode</u>
list the efivars direcrory

> $ `ls /sys/firmware/efi/efivars`

if the command returns the directory without error, then the system is booted in [uefi](https://wiki.archlinux.org/title/UEFI). if the directory doesn't exist you may be booted in [bios](https://wiki.archlinux.org/title/Arch_boot_process#Under_BIOS) or [csm](https://en.wikipedia.org/wiki/Compatibility_Support_Module) mode

----

### <u>Initial Network Setup</u>
check your [network interface](https://wiki.archlinux.org/title/Network_interface#Network_interfaces) is enabled with [iplink](https://man.archlinux.org/man/ip-link.8)

> $ `ip link`

if disabled, check the device driver -- see [ethernet#device-driver](https://wiki.archlinux.org/title/Network_configuration/Ethernet#Device_driver) or [wireless#device-driver](https://wiki.archlinux.org/title/Network_configuration/Wireless#Device_driver)


enter [iwd](https://wiki.archlinux.org/title/Iwctl) interactive prompt

> $ `iwctl`

list all wifi devices

> [iwd]$ `device list`

scan for networks

> [iwd]$ `station <device> scan`

list scanned networks

> [iwd]$ `station <device> get-networks`

finally, connect to specified network

> [iwd]$ `station <device> connect <SSID>`

verify connection
> [iwd]$ `station <device> show`

exit prompt using ctrl+c

----

### <u>System Clock</u>
set system clock with [timedatectl](https://man.archlinux.org/man/timedatectl.1)

> $ `timedatectl set-ntp true`

check status

> $ `timedatectl status`

----

### <u>Disk Partitioning</u>
list disk and block devices 

> $ `lsblk`
 
using your preferred [partitioning](https://wiki.archlinux.org/title/Partition) tool([gdisk](https://wiki.archlinux.org/title/Gdisk), [fdisk](https://wiki.archlinux.org/title/Fdisk), [parted](https://wiki.archlinux.org/title/parted) etc) create the required [root](https://wiki.archlinux.org/title/Root_directory#/)(10GB+) [partition](https://wiki.archlinux.org/title/Root_directory). if booted in uefi, create a [gpt](https://wiki.archlinux.org/title/GPT#GUID_Partition_Table) table and an  [efi](https://wiki.archlinux.org/title/EFI_system_partition)(512MB) [system partition](https://wiki.archlinux.org/title/EFI_system_partition); not necessary for bios with [mbr](https://wiki.archlinux.org/title/MBR#Master_Boot_Record)

if you have an existing efi partition, don't create a new one. do not format it or you will delete all data in the partition, rendering other operating systems unbootable. skip the 'mkfs.vfat' command below and mount the already existing efi partition if available

if you want to create any stacked block devices for [lvm](https://wiki.archlinux.org/title/LVM), [system encryption](https://wiki.archlinux.org/title/Dm-crypt) or [raid](https://wiki.archlinux.org/title/RAID), do it now

----

### <u>Swap Space</u>(Optional)
if you plan to use [swap](https://wiki.archlinux.org/title/Partitioning_tools#Swap) consider creating a [swap partition](https://wiki.archlinux.org/title/Swap#Swap_partition) or [swapfile](https://wiki.archlinux.org/title/Swap#Swap_file) now. your approach would depend on your expectations for your [swap space](https://wiki.archlinux.org/title/Swap_file#Swap_space). to share your swap with other systems or enable hibernation; create a linux swap partition. in comparison, a swapfile can change size on-the-fly and is more easily removed, which may be more desirable for a modestly-sized ssd

if you prefer a swapfile use dd to create one now. the following command will create a 4gb swapfile 

> $ `dd if=/dev/zero of=/swapfile bs=1M count=<4096> status=progress`

a swapfile without the correct permissions is a big security risk. set the file permissions to 600 with [chmod](https://wiki.archlinux.org/title/File_permissions_and_attributes#Changing_permissions)

> $ `chmod 600 /swapfile`

----

#### *_Swap Partition_
if you made a swap partition, format it by replacing 'swap_partition' with it's  assigned block device path, e.g. sda2

> $ `mkswap /dev/<swap_partition>`

then activate it

> $ `swapon /dev/<swap_partition>`

----

#### *_Swapfile_
if you made a swapfile, format it

> $ `mkswap /swapfile`

then activate it

> $ `swapon /swapfile`

----

### <u>Format Partitions</u>
format the root partition you just created with your desired [filesystem](https://wiki.archlinux.org/title/File_systems) by replacing 'root_partition' with it's assigned [block device](https://en.m.wikipedia.org/wiki/Device_file#Block_devices) path, e.g. sda3

> $ `mkfs.<ext4> /dev/<root_partition>`

format replace 'efi_partition' with it's assigned block device path, e.g. sda1

> $ `mkfs.vfat -F32 /dev/<efi_partition>`

----

### <u>Mount Partitions</u>
[mount](https://wiki.archlinux.org/title/Mount) root partition to /mnt

> $ `mount /dev/<root_partition> /mnt`

mount efi [partition](https://wiki.archlinux.org/title/Partitioning#Example_layouts) to [/boot](https://wiki.archlinux.org/title/Partitioning#/boot)

> $ `mount /dev/<efi_partition> /mnt/boot`

----

### <u>Essential Packages</u>
[pacstrap/install](https://man.archlinux.org/man/pacstrap.8) base, [kernel](https://wiki.archlinux.org/title/Kernel) choice & if you have an amd or intel cpu, install your [microcode](https://wiki.archlinux.org/title/Microcode) updates

> $ `pacstrap /mnt base linux linux-firmware nano sudo <cpu_manufacturer>-ucode`

----

### <u>Fstab</u>
generate an [fstab](https://wiki.archlinux.org/title/Fstab) file from detected mounted block devices, defined by labels

> $ `genfstab -L /mnt >> /mnt/etc/fstab`

check generated fstab

> $ `cat /mnt/etc/fstab`

----

### <u>Change Root</u>
[chroot](https://wiki.archlinux.org/title/Change_root) into freshly installed system

> $ `arch-chroot /mnt`

----

### <u>Time Zone</u>
set [time zone](https://wiki.archlinux.org/title/Time_zone)

> $ `ln -sf /usr/share/zoneinfo/<region>/<city> /etc/localtime`

generate /etc/adjtime with [hwclock](https://man.archlinux.org/man/hwclock.8)

> $ `hwclock --systohc`

this assumes the hardware clock is set to [utc](https://en.m.wikipedia.org/wiki/UTC). for more, see [system time#time standard](https://wiki.archlinux.org/title/System_time#Time_standard)

----

### <u>Localization</u>
[edit](https://wiki.archlinux.org/title/Textedit) /etc/locale.gen and un[comment](https://linuxhint.com/bash_comments/) 'en_US.UTF-8 UTF-8' or any other necessary [locales](https://wiki.archlinux.org/title/Locale) by removing the #

> $ `nano /etc/locale.gen`

use ctrl+x then 'y' to save and close [nano](https://wiki.archlinux.org/title/Nano). if you want to use a different editor -- see [documents#editors](https://wiki.archlinux.org/title/List_of_applications/Documents#Text_editors)

generate the locales

> $ `locale-gen`

[create](https://wiki.archlinux.org/title/Textedit) the [locale.conf](https://man.archlinux.org/man/locale.conf.5) file and set the [system locale](https://wiki.archlinux.org/title/Locale#Setting_the_system_locale) 

> $ `echo LANG=en_US.UTF-8 >> /etc/locale.conf`

----

### <u>Network Configuration</u>
create [hostname](https://wiki.archlinux.org/title/Hostname) file

> $ `echo <hostname> >> /etc/hostname`

add matching entries to [hosts](https://man.archlinux.org/man/hosts.5)

> $ `nano /etc/hosts`
```diff
127.0.0.1    localhost  
::1          localhost  
127.0.1.1    <hostname>.localdomain <hostname>
```

if the system has a permanently assigned ip address, use it instead of '127.0.1.1'

----

#### *_Systemd-Networkd_
install any desired [network managment](https://wiki.archlinux.org/title/Network_configuration) software. my preferred method is [systemd-networkd](https://wiki.archlinux.org/title/Systemd-networkd). for an easier, configureless setup -- check out networkmanager below 

connect to the network with [wpa_passphrase](https://wiki.archlinux.org/title/Wpa_supplicant#Connecting_with_wpa_passphrase)

> $ `wpa_passphrase <ssid> <password> > /etc/wpa_supplicant/wpa_supplicant-<interface>.conf`

[enable & start](https://wiki.archlinux.org/title/Start/enable) the wpa_supplicant daemon reading the config file just created

> $ `systemctl enable --now wpa_supplicant@<interface>`

setup systemd for wireless networking. for ethernet -- see [#wired adapter](https://wiki.archlinux.org/title/Systemd-networkd#Wired_adapter_using_DHCP). for a static connection -- see [#static](https://wiki.archlinux.org/title/Systemd-networkd#Wired_adapter_using_a_static_IP)

> $ `nano /etc/systemd/network/25-wireless.network`
```diff
[Match]
Name=<interface>

[Network] 
DHCP=yes
```
enable and start the systemd-network service daemon

> $ `systemctl enable --now systemd-networkd`

----

#### *_NetworkManager_
install [networkmanager](https://wiki.archlinux.org/title/NetworkManager)

> $ `pacman -S NetworkManager`

enable and start it

> $ `systemctl enable --now NetworkManager`

----

### <u>Initramfs</u>
creating an initramfs image isn't necessary since [mkinitcpio](https://wiki.archlinux.org/title/Mkinitcpio) was ran when pacstrap installed the kernel

for [lvm](https://wiki.archlinux.org/title/Install_Arch_Linux_on_LVM#Adding_mkinitcpio_hooks), [system encryption](https://wiki.archlinux.org/title/Dm-crypt) or [raid](https://wiki.archlinux.org/title/RAID#Configure_mkinitcpio) modify [mkinitcpio.conf](https://man.archlinux.org/man/mkinitcpio.conf.5) then recreate the initramfs image

> $ `mkinitcpio -P`

----

### <u>Users & Passwords</u>
create a new user

> $ `useradd -m <username>`

add created user to the wheel group

> $ `usermod -aG wheel <username>`

uncomment '%wheel', by removing the #

> $ `EDITOR=nano visudo`

set created user [password](https://wiki.archlinux.org/title/Password)

> $ `passwd <username>`

set [root user](https://wiki.archlinux.org/title/Root_user) password

> $ `passwd`

optionally, disable access to [superuser/root](https://en.m.wikipedia.org/wiki/Root_user), locking password entry for root user. this will give your system increased security and you can still elevate a user in the wheel group to superuser priveleges with [sudo](https://wiki.archlinux.org/title/Sudo) and [su](https://wiki.archlinux.org/title/Su) commands

> $ `passwd -l root`

----

### <u>Boot Loader</u>
install a linux-capable [boot loader](https://wiki.archlinux.org/title/Boot_loader). for simplicity and ease of use I recommend systemd-boot, not grub for uefi. systemd-boot is not compatible with systems booted in legacy bios mode. systemd-boot will boot any configured efi image including windows.

do not install two different boot loaders on the same system. it will cause many conflicts. if you are familiar with grub and it works for you, skip systemd-boot or vice-versa

----

#### *_Systemd-boot_

install [systemd-boot](https://wiki.archlinux.org/title/Systemd-boot) in the efi system partition

> $ `bootctl --path=/boot install`

add a [boot entry](https://wiki.archlinux.org/title/Systemd-boot#Adding_loaders) and load installed microcode updates

> $ `nano /boot/loader/entries/<entry>.conf`
```diff
title <Arch Linux>  
linux /vmlinuz-linux  
initrd /<cpu_manufacturer>-ucode.img  
initrd /initramfs-linux.img  
options root=/dev/<root_partition> rw quiet log-level=0
```

if you installed a different kernel, such as linux-zen, you would add '-zen' above to 'vmlinuz-linux' and 'initframs-linux'. this will boot the system using your selected kernel. for more -- see [kernel paramters](https://wiki.archlinux.org/title/Kernel_parameters#systemd-boot) 

edit [loader config](https://man.archlinux.org/man/loader.conf.5)

> $ `nano /boot/loader/loader.conf`
```diff
default <entry>.conf  
timeout <3>  
console-mode <max>  
editor <no>
```

verify entry is bootable

> $ `bootctl list`

----

#### *_GRUB_
install [grub](https://wiki.archlinux.org/title/GRUB) but only install [efibootmgr](https://wiki.archlinux.org/title/EFISTUB#efibootmgr) if you are booted in uefi mode. also, install  [os-prober](https://archlinux.org/packages/?name=os-prober) to automatically add boot entries for other operating systems 

> $ `pacman -S grub efibootmgr os-prober`

##### *_for systems booted in uefi mode use:_

> $ `grub-install --target=x86_64-efi --efi-directory=/boot/grub --bootloader-id=GRUB`

##### *_othwerwise, if booted in bios mode use this; where path is the entire disk, not just a partition or path to a directory_

> $ `grub-install --target=i386-pc /dev/sdx`

generate the [grub config](https://wiki.archlinux.org/title/GRUB#Configuration) file. this will automatically detect your arch linux installation

> $ `grub-mkconfig -o /boot/grub/grub.cfg`

if you get a warning that os-prober will not be executed.. uncomment 'GRUB_DISABLE_OS_PROBER=false' by removing the #

> $ `nano /etc/default/grub`

----

### <u>End</u>
exit chroot using ctrl+d

unmount all partitions; check if busy

> $ `unmount -R /mnt`

reboot the system

> $ `reboot`
