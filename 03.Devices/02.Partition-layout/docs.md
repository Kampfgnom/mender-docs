---
title: Partition layout
taxonomy:
    category: docs
---

##Overview

In order to support robust rollback, Mender requires the device to have a certain partition layout.
At least four different partitions are needed:
* one boot partition, containing the U-Boot bootloader and its environment
* two partitions for storing the root file system and kernel. The kernel image file, zImage, and any device tree binary should be stored in directory /boot
* one for persistent data

One of the rootfs and kernel partitions will be marked as the *active* partition, from which the kernel and rootfs will be booted.
The other, called the *inactive* partition, will be used by the update mechanism to write the updated image.
After an update their roles are swapped.

The persistent data partition stores data that needs to be preserved through an update.

A sample partition layout is shown below:

![Mender client partition layout](mender_client_partition_layout.png)


##Flash memory types

Embedded devices almost universally use flash memory for storage.
An important property of flash memory cells is that they can only
handle a certain amount of writes until they fail (wear out).
Wear leveling (distributing writes across the cells)
and error correction (avoiding use of failed cells) should be carried out
in order to prolong their life.

However, these tasks can be handled by the flash device itself
or by software (OS and file system). From the software's point of view,
there are two types of flash memory:

* **Block device.** These flash devices will expose a linear array of
blocks to the OS, just like hard drives do. This is the most common
type of flash device used with Linux, except in very low-cost or older embedded devices.
They are generally easy to work with and you can put block-device file systems,
like ext4 and fat, directly on top of them. Internally these devices contain
a *memory controller* that runs a Flash Translation Layer firmware that implements
wear leveling and error correction, so this is taken care of transparently. For this
reason, they are also sometimes referred to as **Flash Translation Layer (FTL) devices**.
For example, these types of flash devices expose themselves as block devices: **SD, mini-SD, micro-SD,
MMC, eMMC, RS-MMC, SSD, USB, CompactFlash, MemoryStick, MemoryStick Micro**.

* **Raw flash.** Raw flash devices do not have a memory controller that takes
care of wear leveling or error correction, so this *must be handled in
software*. In Linux, raw flash devices are exposed as a **Memory Technology
Device (MTD)** file. Care must be taken when selecting a file system to ensure
that it is MTD-aware and properly handles wear leveling and error correction.
Popular file systems for MTD devices include UBIFS, JFFS2, and YAFFS.
Consult the [raw flash](../raw-flash) section for details on setting up and
configuration.


##File system types

When [building a Mender Yocto Project image](../../artifacts/building-mender-yocto-image) the build output in `tmp/deploy/images/<MACHINE>` includes a binary rootfs file system image (e.g. with `.mender` extension), as well as a complete disk image (with `.sdimg` extension). The binary rootfs file system images are used when deploying updates to the device, while the `.sdimg` image is typically used just once during initial device provisioning to flash the entire storage, and includes the partition layout and all partitions.

In general Mender does not have dependencies on a specific file system type as long as it is for a [block device](#flash-memory-types), but the version of U-Boot you are using must support the file system type used for rootfs because it needs to read the Linux kernel from the file system and start the Linux boot process.

The standard Yocto Project `IMAGE_FSTYPES` variable will be used to determine the image types to create in Yocto deploy directory. The meta-mender layer will add the `mender` type to that variable, and usually `sdimg`, `uefiimg` or a different type ending with `img`, depending on which [image features](../../artifacts/image-configuration/features#list-of-features) are enabled. The filesystem used for the individual partition files will be based on the `ARTIFACTIMG_FSTYPE` variable.  It is advised that you clean up the `IMAGE_FSTYPES` variable to avoid creating unnecessary image files.


##Configuring storage

In order to select the storage device where the partitions are expected to be located on the device, `MENDER_STORAGE_DEVICE` should be set, either in `machine.conf` or in `local.conf`. The value should be the raw device containing the entire storage, not any single partition. For example:

```bash
MENDER_STORAGE_DEVICE = "/dev/mmcblk0"
```

! For memory card storage and some other types of storage, the default way to refer to partitions is to add a "p" and then the number of the partition (for example `/dev/mmcblk0p1`). If you're using a storage type which doesn't follow this scheme (for example `/dev/sda1`), then you also need to set `MENDER_STORAGE_DEVICE_BASE` to the correct value, such as `MENDER_STORAGE_DEVICE_BASE = "${MENDER_STORAGE_DEVICE}"`. The default is the value of `MENDER_STORAGE_DEVICE` plus a "p".


###More detailed storage configuration

If you need more fine grained control over which partitions Mender will use, you can set one or more the following variables to specific partition strings:

* `MENDER_BOOT_PART`
* `MENDER_DATA_PART`
* `MENDER_ROOTFS_PART_A`
* `MENDER_ROOTFS_PART_B`

For example:

```bash
MENDER_BOOT_PART = "${MENDER_STORAGE_DEVICE_BASE}1"
MENDER_DATA_PART = "${MENDER_STORAGE_DEVICE_BASE}5"
MENDER_ROOTFS_PART_A = "${MENDER_STORAGE_DEVICE_BASE}2"
MENDER_ROOTFS_PART_B = "${MENDER_STORAGE_DEVICE_BASE}3"
```

!! Note that the Mender image builder will not produce such images, so only set these variables if you're building partitioned images yourself, with a different layout than the default Mender layout (the example above reflects the default).


##Configuring the partition sizes

When [building a Mender Yocto Project image](../../artifacts/building-mender-yocto-image) Mender defines and uses certain OpenEmbedded variables which are used to define the sizes of the partitions.

| Mount point  | Purpose                                                 | Default size | Variable to configure size     |
|--------------|---------------------------------------------------------|--------------|--------------------------------|
| `/`          | Store the root file system and kernel.                  | auto         | `MENDER_STORAGE_TOTAL_SIZE_MB` |
| &lt;BOOT&gt; | Store the bootloader.                                   | 16 MB        | `MENDER_BOOT_PART_SIZE_MB`     |
| `/data`      | Store persistent data, preserved during Mender updates. | 128 MB       | `MENDER_DATA_PART_SIZE_MB`     |

The value of &lt;BOOT&gt; depends on what features are enabled:
* If `mender-uboot` is enabled: `/uboot`
* If `mender-grub` and `mender-bios` are enabled: `/boot/grub`
* If only `mender-grub` is enabled: `/boot/efi`

You can override these default values in your `local.conf`. For details consult [Mender image variables](../../artifacts/variables).


##Preserving data and configuration across updates

As Mender does a full rootfs image update, care must be taken in where persistent data is stored. The contents of the partition mounted on `/data` is preserved across updates. In fact, the Mender client itself uses `/data/mender` as a backing to store data that needs to be kept across updates.

If you have data or configuration that you need to preserve across updates, the recommended approach is to create a symlink from where it gets written to somewhere within `/data/`. For example, if you have an application that writes to `/etc/application1`, then you can create a symlink `/etc/application1` -> `/data/application1` to ensure the data it writes is not lost during a Mender rootfs update.

##Deploying files to the persistent data partition

When [building a Mender Yocto Project image](../../artifacts/building-mender-yocto-image), if you need to include files in the persistent data partition, all you have to do is add those files to the `/data` directory in the root filesystem, and the files will automatically be included on the data partition. For example:

```bash
do_install() {
    install -d ${D}/data
    install -m 0644 persistent.txt ${D}/data/
}
```

A sample recipe (`hello-mender`) is included in the `meta-mender-demo` layer which deploys a text file to the persistent data partition.

! Keep in mind that any files you add to the `/data` directory will not be included in `.mender` artifacts, since they don't contain a data partition. Only complete partitioned images (`.biosimg`, `.sdimg`, `.uefiimg`, etc) will contain the files.

!!! In Yocto Project 2.4 rocko and earlier, there was a different method for adding files to the data partition. Please see the [`MENDER_DATA_PART_DIR` variable](../../artifacts/variables#mender_data_part_dir) for details on this now obsolete method.


## Producing a standalone data partition image

Although it is not needed for most work with Mender, for some flashing setups, it may be useful to have the sole data partition available as an image file. If this is needed, simply add `dataimg` to the Yocto Project `IMAGE_FSTYPES` variable, and the image file will be built and given the `.dataimg` suffix. Its filesystem type will be the value of [`ARTIFACTIMG_FSTYPE`](../../artifacts/variables#artifactimg_fstype). For example:

```bash
IMAGE_FSTYPES_append = " dataimg"
```
