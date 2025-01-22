
---
title: Hetzner debian server with Root-on-ZFS
layout: default
---

{: .note }
This guide will help you setup a server (works on both cloud and dedicated) with Debian 12, Root-on-ZFS filesystem and ZFSBootMenu as GRUB alternative.

{: .danger }
If you are following this tutorial on Hetzner Cloud virtual machines, please note that not all instance types are equipped with UEFI. If you want to make sure you get an UEFI-enabled instance, please select one of the `dedicated vCPU` instance types.

## Reboot your server into recovery system

## Partition, format and mount your disk(s) 
I recommend using `cfdisk` utility for managing partitions.

This tutorial was created using a dedicated server with NVME disks. If you are working with other disk types or on an Hetzner Cloud instance, you need to adjust your `/dev/...` device names accordingly.

If your server has more than one disk, you need to format all of them with the following layout:

1. small partition (100MB) with type `EFI system`
2. rest of the disk default `Linux Filesystem`

```shell
# repeat for additional disks if required
mkfs.vfat -F32 /dev/nvme0n1p1

zpool create \
-o compatibility=openzfs-2.1-linux \
-o autotrim=on \
-O compression=lz4 \
-O acltype=posixacl \
-O xattr=sa \
-O relatime=on \
-O canmount=off \
-O mountpoint=none \
data /dev/nvme0n1p2

zfs create -o mountpoint=/ -o canmount=noauto data/root

zpool set bootfs=data/root data

zpool export data
zpool import -N -R /mnt data && zfs mount data/root
```

## Install base debian system

```shell
debootstrap bookworm /mnt

for i in /dev /dev/pts /proc /sys /sys/firmware/efi/efivars /run; do mount -B $i /mnt$i; done

# repeat if you have multiple disks to make all of them bootable later
mkdir -p /mnt/boot/efi1

# get block IDs of our created EFI partitions
blkid | grep vfat

# replace XXX with the ID from above, duplicate mount if you have multiple disks
echo "
UUID=XXXX-XXXX  /boot/efi1      vfat    umask=0022,fmask=0022,dmask=0022      0       1
" >> /mnt/etc/fstab

echo "
deb http://deb.debian.org/debian bookworm contrib main non-free-firmware
deb http://deb.debian.org/debian bookworm-updates contrib main non-free-firmware
deb http://deb.debian.org/debian-security bookworm-security contrib main non-free-firmware
" > /mnt/etc/apt/sources.list

cp /etc/hosts /mnt/etc/hosts

chroot /mnt

# IMPORTANT!
passwd

# adjust /etc/resolv.conf to your liking

# adjust /etc/network/interfaces to your network setup
echo "
auto eno1
iface eno1 inet static
  address xx.xx.xx.xx
  netmask 255.255.255.255
  gateway xx.xx.xx.xx
iface eno1 inet6 static
  address 2a01::xxxxxxxx
  netmask 64
  gateway fe80::1
" > /etc/network/interfaces

apt update && apt install -y locales && dpkg-reconfigure locales && apt install -y efibootmgr wget console-setup openssh-server

# enable PermitRootLogin in /etc/ssh/sshd_config

apt install -y linux-image-amd64 zfs-initramfs dosfstools

update-initramfs -c -k all
```

## Install ZFSBootManager on all EFI partitions

```shell
mount -a

wget https://github.com/zbm-dev/zfsbootmenu/releases/download/v2.3.0/zbm-kcl && chmod +x zbk-kcl

# repeat for all disks that have EFI partition
mkdir -p /boot/efi1/EFI/ZBM
wget https://get.zfsbootmenu.org/efi -O /boot/efi1/EFI/ZBM/VMLINUZ.EFI
./zbm-kcl -a zbm.skip /boot/efi1/EFI/ZBM/VMLINUZ.EFI
efibootmgr -c -d "/dev/nvme0n1" -p "1" \
  -L "ZFSBootMenu" \
  -l '\EFI\ZBM\VMLINUZ.EFI'

# IMPORTANT! change boot order, PXE boot at first position
efibootmgr -o x,x,x...
```

## Additional configuration of our new OS
```shell
# clone server utils scripts
apt install -y git
git clone https://github.com/janxb/serverutils.git /usr/local/sbin

apt install -y bash-completion
echo "
if [ -f /etc/bash_completion ]; then
  . /etc/bash_completion
fi
" >> ~/.bashrc

# automatic upgrades for all packages
apt install -y unattended-upgrades
echo '
APT::Periodic::Enable "1";
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Download-Upgradeable-Packages "1";
APT::Periodic::Unattended-Upgrade "1";
APT::Periodic::AutocleanInterval "14";
APT::Periodic::Verbose "0";
' > /etc/apt/apt.conf.d/51unattended-upgrades-custom-config
echo '
Unattended-Upgrade::Origins-Pattern {
  "origin=*";
};
' > /etc/apt/apt.conf.d/52unattended-upgrades-custom-origins
```

## Final steps
```shell
exit
umount -n -R /mnt
reboot
```
