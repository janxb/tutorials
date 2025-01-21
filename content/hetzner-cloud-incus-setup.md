---
title: Incus node setup on Hetzner Cloud
layout: default
---

{: .note }
This guide will help you setup an [Incus](https://linuxcontainers.org/incus/) node with Debian 12 OS and an external Cloud volume as ZFS storage pool. It can also be perfectly combined with my [Root-on-ZFS tutorial](/hetzner-debian-root-on-zfs.html).

{: .danger }
As every other tutorial on here, it requires THINKING YOURSELF and not blindly copying the stuff you find here. And now: Have fun!

# Partition your disk
You can either boot into a custom GParted ISO if you like GUI tools, or boot into recovery OS.

In this tutorial we are using the smallest disk size, because our data will live on an external Cloud volume. This setup requires additional partitions to speed up our ZFS storage pool.

{: .info }
If you are combining this tutorial with my Root-on-ZFS tutorial, please keep in mind that you also need to create a small EFI partition.

We want the following partition layout:
1. 20GB system partition
2. 3GB partition for SLOG
3. 57GB (rest of disk) for L2ARC

# Install Incus from Zabbly repository
https://github.com/zabbly/incus

# Configure Incus
Please select correct option for ZFS pool setup. If you are using Cloud volume, select:
- yes, setup new ZFS pool
- yes, use external block device
- provide path to cloud volume, for me it was `/dev/sdb`

```shell
incus admin init
```

# Customize Incus environment
```shell
apt install git lz4 -y
git clone https://github.com/janxb/serverutils /usr/local/sbin/
set-timezone Europe/Berlin
echo br_netfilter >> /etc/modules-load.d/modules.conf

# if you are installing incus on existing (local) pool, adjust pool name as required
export POOL=default

incus config set images.compression_algorithm lz4
incus config set backups.compression_algorithm lz4
zfs set compression=lz4 $POOL
zpool set autoexpand=on $POOL
zpool set autotrim=on $POOL

# add special devices to your pool, adjust sdXX as required
zpool add $POOL cache /dev/sdXX # large 57GB L2ARC partition
zpool add $POOL log /dev/sdXX # small 3GB SLOG partition

#  skip if you don't want to reach your incus node via web UI
incus config set core.https_address :8443

# custom volumes for images and backups, otherwise they will be put on local disk
incus storage volume create $POOL backups
incus storage volume create $POOL images
incus config set storage.backups_volume $POOL/backups
incus config set storage.images_volume $POOL/images

# scheduled daily snapshots for all containers
incus profile set default snapshots.pattern "{{ creation_date|date:'2006-01-02_15-04-05' }}"
incus profile set default snapshots.schedule "@daily"
incus profile set default snapshots.schedule.stopped "true"
incus profile set default snapshots.expiry "1w"
```

# Final notes
In one of the earlier steps, you checked out my scripts GIT repository at https://github.com/janxb/serverutils.

This repo also contains some useful scripts for managing your incus installation, for example:

**zfs-iotop:** view disk activity for your ZFS pools

**zfs-list:** view ZFS datasets with some extended fields

**incus-backup-zfs:** export/import incus containers to native ZFS files

**incus-network-forward:** manage port forwards to your incus containers