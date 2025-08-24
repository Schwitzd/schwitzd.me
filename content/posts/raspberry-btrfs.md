+++
title = 'Raspberry Btrfs'
date = 2024-07-10T15:52:10Z
draft = false
+++

I would like to build a simple NAS using my Raspberry Pi 5, equipped with the [Geekworm X1011](https://wiki.geekworm.com/X1011) and an M.2 NVMe drives. This project will also provide me with the opportunity to use [BTRFS](https://en.wikipedia.org/wiki/Btrfs) for the first time.

## Getting started

1. To install BTRFS excecute the following commnad:

    ```sh
    sudo apt install btrfs-progs
    ```

1. Uses `lsblk` to get the current partitions status

    ```sh
    $ lsblk 
    NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
    mmcblk0     179:0    0  14.9G  0 disk 
    ├─mmcblk0p1 179:1    0   512M  0 part /boot/firmware
    └─mmcblk0p2 179:2    0  14.4G  0 part /
    nvme0n1     259:0    0 476.9G  0 disk 
    └─nvme0n1p1 259:1    0 476.9G  0 part 
    nvme1n1     259:2    0 238.5G  0 disk 
    nvme2n1     259:7    0 238.5G  0 disk 
    ```

1. If necessary, use one of the many tools, such as `wipefs`, `fdisk`, `cfdisk`, or `parted`, to remove the old partitions, for example:

    ```sh
    sudo wipefs -a /dev/<disk-name>
    ```

## BTFS Pool

I'm going to create a poll of discs, because at the time of writing I have 3x M.2 of different sizes, I decided not to use any RAID as I'll be wasting some space that I don't want. So I take the risk of losing data.
The best way to use RAID is to have all the discs of the same size, otherwise the usable space for each disc is limited to the size of the smallest disc in the array.

1. Create the mount point of the pool

    ```sh
    sudo mkdir /mnt/my_pool
    sudo chown -R $USER:$USER /mnt/my_pool
    ```

1. Create the BTRFS pool, in my case with the three disks

    ```sh
    sudo mkfs.btrfs -d single -L my_pool /dev/nvme0n1 /dev/nvme1n1 /dev/nvme2n1 -f
    ```

1. Get your nearly created pool details

    ```sh
    sudo btrfs filesystem show
    ```

To get details about the BTRS pool

```sh
sudo btrfs filesystem usage /mnt/my_pool
```

### Expand the pool

1. If necessary, wipe the disk before adding it to the pool.
1. Add the disk to the existing Btrfs pool

    ```sh
    sudo btrfs device add /dev/nvme3n1 /mnt/btrfs_pool
    ```

1. Rebalance the data across all disks

    ```sh
    sudo btrfs balance start /mnt/btrfs_pool

    # to check the rebalance status of a Btrfs volume
    sudo btrfs balance status /mnt/my_pool
    ```

    Balancing a Btrfs pool is a **computationally intensive operation** and can take a significant amount of time depending on the **size and usage** of the pool.

### Replace a disk

For various reasons you may need to replace a disk in your Btrfs pool: a drive might fail, or you might simply want to upgrade to a larger one.

1. Identify the disk you want to replace by checking its serial number to avoid accidentally removing the wrong drive:

    ```sh
    lsblk -d -o NAME,SIZE,MODEL,SERIAL
    ```

1. If the disk is still working and visible in the pool, remove it cleanly:

    ```sh
    sudo btrfs device remove /dev/nvme3n1 /mnt/my_pool/
    ```

    If the disk has already failed and is missing, tell Btrfs to forget it:

    ```sh
    sudo btrfs device remove missing /mnt/my_pool
    ```

Once you have physically replaced the disk, follow the chapter [Expand the pool](#expand-the-pool) to add the new device to the existing pool.

## Mount the pool

Now is time to mount your pool in `/etc/fstab`, get the **UUID** from the previus command and add following line

```sh
# My Pool mount options
MOUNT_OPTIONS="autodefrag,compress=zstd:3,discard=async,noatime,nodiratime,nodev,rw,space_cache=v2,ssd,user"

# Update fstab with new mount point
echo "UUID=<YOUR-UUID> /mnt/my_pool btrfs $MOUNT_OPTIONS 0 0" | sudo tee -a /etc/fstab
```

Some details about mount options:

* `autodefrag`: enable automatic file defragmentation for small random writes in files with a maximum file size of 64K
* `compress=zstd:3`: enables lossless Zstandard compression at level 3, focusing on a balance between speed and compression ratio.
* `discard=async`: enables asynchronous TRIM operations, improving SSD performance by deferring TRIM commands to run in the background
* `noatime`: allows measurable performance gains by eliminating the need for the system to write to the file system for files that are simply read
* `nodiratime`: prevents updating directory access times, improving performance by reducing unnecessary write operations
* `nodev`: disallows creating and accessing device nodes (used in particular for special files in /dev);
* `noexec`: does not allow the execution of executable binaries in the mounted file system
* `nosuid`: specifies that the filesystem cannot contain set userid files
* `rw`: allows reading and writing
* `space_cache=v2`: control the free space cache. This greatly improves performance when reading block group free space into memory
* `ssd`: by default, Btrfs will enable or disable SSD allocation heuristics depending on whether a rotational or non-rotational device is in use
* `user`: allows ordinary users to mount and unmount the filesystem
