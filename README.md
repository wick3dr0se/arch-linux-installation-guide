# Arch Linux [Installation](https://wiki.archlinux.org/title/Installation_guide) Guide
#### The goal of this guide is to provide an easier to interpret, while still chomprehensive how-to for installing Arch Linux on x86_64 architecture devices. This guide primarily focuses on utilizing systemd to keep it as close to minimal as possible; there are refrences if you prefer a different network tool or boot loader.
----

### Verify Boot Mode
list the efivars direcrory

> $ `ls /sys/firmware/efi/efivars`

if the command returns the directory without error, then the system is booted in [uefi](https://wiki.archlinux.org/title/UEFI). if the directory doesn't exist you may be booted in [bios](https://wiki.archlinux.org/title/Arch_boot_process#Under_BIOS) or [csm](https://en.wikipedia.org/wiki/Compatibility_Support_Module).

### Initial Network Setup
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

### System Clock
set system clock with [timedatectl](https://man.archlinux.org/man/timedatectl.1)

> $ `timedatectl set-ntp true`

check status

> $ `timedatectl status`

### Disk Partitioning
[list disks/block](https://wiki.archlinux.org/title/Lsblk) devices 

> $ `lsblk`
 
using your preferred [partitioning](https://wiki.archlinux.org/title/Partition) tool([gdisk](https://wiki.archlinux.org/title/Gdisk), [fdisk](https://wiki.archlinux.org/title/Fdisk), [parted](https://wiki.archlinux.org/title/parted) etc) create the required [root](https://wiki.archlinux.org/title/Root_directory#/)(10GB+) [partition](https://wiki.archlinux.org/title/Root_directory). if booted in uefi with [gpt](https://wiki.archlinux.org/title/GPT#GUID_Partition_Table), create an  [efi](https://wiki.archlinux.org/title/EFI_system_partition)(512MB) [system partition](https://wiki.archlinux.org/title/EFI_system_partition); not necessary for bios with [mbr](https://wiki.archlinux.org/title/MBR#Master_Boot_Record). optionally create a [swap](https://wiki.archlinux.org/title/Swap) partition

if you have an existing efi partition, don't create one. do not format it or you will delete all data in the partition, rendering other operating systems unbootable. skip the 'mkfs.vfat' command below and mount the already existing efi partition if available. 

if you want to create any stacked block devices for [lvm](https://wiki.archlinux.org/title/LVM), [system encryption](https://wiki.archlinux.org/title/Dm-crypt) or [raid](https://wiki.archlinux.org/title/RAID), do it now

### Format Partitions
replace 'root_partition' with it's assigned [block device](https://en.m.wikipedia.org/wiki/Device_file#Block_devices), e.g. sda3

> $ `mkfs.ext4 /dev/<root_partition>`

replace 'efi_partition' with it's assigned block device, e.g. sda1

> $ `mkfs.vfat -F32 /dev/<efi_partition>`

if you made a swap partition. replace 'swap_partition' with it's assigned block device, e.g. sda2

> $ `mkswap /dev/<swap_partition>`

> $ `swapon /dev/<swap_partition>`

### Mount Partitions
[mount](https://wiki.archlinux.org/title/Mount) root partition to /mnt

> $ `mount /dev/<root_partition> /mnt`

mount efi [partition](https://wiki.archlinux.org/title/Partitioning#Example_layouts) to [/boot](https://wiki.archlinux.org/title/Partitioning#/boot)

> $ `mount /dev/<efi_partition> /mnt/boot`

### Essential Packages
[pacstrap/install](https://man.archlinux.org/man/pacstrap.8) base, [kernel](https://wiki.archlinux.org/title/Kernel) choice & if you have an amd or intel cpu, install your [microcode](https://wiki.archlinux.org/title/Microcode) updates

> $ `pacstrap /mnt base linux linux-firmware nano sudo <cpu_manufacturer>-ucode wpa_supplicant`

### Fstab
generate an [fstab](https://wiki.archlinux.org/title/Fstab) file from detected mounted block devices, defined by labels

> $ `genfstab -L /mnt >> /mnt/etc/fstab`

check generated fstab

> $ `cat /mnt/etc/fstab`

### Change Root
[chroot](https://wiki.archlinux.org/title/Change_root) into freshly installed system

> $ `arch-chroot /mnt`

### Time Zone
set [time zone](https://wiki.archlinux.org/title/Time_zone)

> $ `ln -sf /usr/share/zoneinfo/<region>/<city> /etc/localtime`

generate /etc/adjtime with [hwclock](https://man.archlinux.org/man/hwclock.8)

> $ `hwclock --systohc`

this assumes the hardware clock is set to [utc](https://en.m.wikipedia.org/wiki/UTC). for more, see [system time#time standard](https://wiki.archlinux.org/title/System_time#Time_standard)

### Localization
[edit](https://wiki.archlinux.org/title/Textedit) /etc/locale.gen and un[comment](https://linuxhint.com/bash_comments/) 'en_US.UTF-8 UTF-8' or any other necessary [locales](https://wiki.archlinux.org/title/Locale)

> $ `nano /etc/locale.gen`

generate the locales

> $ `locale-gen`

[create](https://wiki.archlinux.org/title/Textedit) the [locale.conf](https://man.archlinux.org/man/locale.conf.5) file and set [LANG variable](https://wiki.archlinux.org/title/Locale#Setting_the_system_locale) 

> $ `echo LANG=en_US.UTF-8 >> /etc/locale.conf`

### Network Configuration
create [hostname](https://wiki.archlinux.org/title/Hostname) file

> $ `echo <hostname> >> /etc/hostname`

add matching entries to [hosts](https://man.archlinux.org/man/hosts.5)

> $ `nan.o /etc/hosts`
```diff
127.0.0.1    localhost  
::1          localhost  
127.0.1.1    <hostname>.localdomain <hostname>
```

if the system has a permanently assigned ip address, use it instead of '127.0.1.1'

install any desired [network managment](https://wiki.archlinux.org/title/Network_configuration) software. in this guide, we're using systemd-networkd. for an easier, configureless setup -- check out [networkmanager](https://wiki.archlinux.org/title/NetworkManager)

connect to the network with [wpa_passphrase](https://wiki.archlinux.org/title/Wpa_supplicant#Connecting_with_wpa_passphrase)

> $ `wpa_passphrase <ssid> <password> > /etc/wpa_supplicant/wpa_supplicant-<interface>.conf`

[enable & start](https://wiki.archlinux.org/title/Start/enable) the wpa_supplicant daemon reading the config file just created

> $ `systemctl enable --now wpa_supplicant@<interface>`

setup systemd for wireless networking. for ethernet -- see [wired adapter](https://wiki.archlinux.org/title/Systemd-networkd#Wired_adapter_using_DHCP)

> $ `nano /etc/systemd/network/25-wireless.network`
```diff
[Match] Name=wlp2s0

[Network] DHCP=yes
```

enable & start the systemd-network service daemon

> $ `systemctl enable --now systemd-networkd`

### Initramfs
creating an initramfs image isn't necessary since [mkinitcpio](https://wiki.archlinux.org/title/Mkinitcpio) was ran when pacstrap installed the kernel

for [lvm](https://wiki.archlinux.org/title/Install_Arch_Linux_on_LVM#Adding_mkinitcpio_hooks), [system encryption](https://wiki.archlinux.org/title/Dm-crypt) or [raid](https://wiki.archlinux.org/title/RAID#Configure_mkinitcpio) modify [mkinitcpio.conf](https://man.archlinux.org/man/mkinitcpio.conf.5) then recreate the initramfs image

> $ `mkinitcpio -P`

### Users & Passwords
create a new user

> $ `useradd -m <username>`

add created user to the wheel group for sudo priveleges

> $ `usermod -aG wheel <username>`

uncomment %wheel

> $ `EDITOR=nano visudo`

set created user [password](https://wiki.archlinux.org/title/Password)

> $ `passwd <username>`

set [root user](https://wiki.archlinux.org/title/Root_user) password

> $ `passwd`

optionally, disable access to [superuser](https://en.m.wikipedia.org/wiki/Root_user)/root user for security

> $ `passwd -l root`

### Boot Loader
install a linux-capable [boot loader](https://wiki.archlinux.org/title/Boot_loader); in our case [systemd-boot](https://wiki.archlinux.org/title/Systemd-boot)

install the boot loader in the efi system partition

> $ `bootctl --path=/boot install`

add a [boot entry](https://wiki.archlinux.org/title/Systemd-boot#Adding_loaders) and load installed microcode updates

> $ `nano /boot/loader/entries/<entry>.conf`
```diff
title Arch Linux  
linux /vmlinuz-linux  
initrd /<cpu_manufacturer>-ucode.img  
initrd /initramfs-linux.img  
options root=/dev/<root_partition> rw quiet log-level=0
```

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

### End
exit chroot using ctrl+d

unmount all partitions; check if busy

> $ `unmount -R /mnt`

reboot the system

> $ `reboot`
