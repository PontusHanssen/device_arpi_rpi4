# Android Automotive Raspberry Pi 4B

## Hardware

* Raspberry Pi 4B
* 7" HDMI+USB Touchscreen
* PiCAN2 hat
* OBD-II to D-SUB adapter

Start by reading : [https://github.com/PontusPersson/local_manifests](https://github.com/PontusPersson/local_manifests)

# Install prerequisites

```
$ sudo apt install gcc-aarch64-linux-gnu libssl-dev flex build-essential bison
```


## ARM64 Toolchain

To compile the kernel we need a cross-compile toolchain for arm64.

```
$ cd $HOME
$ wget https://releases.linaro.org/components/toolchain/binaries/7.5-2019.12/aarch64-linux-gnu/gcc-linaro-7.5.0-2019.12-x86_64_aarch64-linux-gnu.tar.xz
$ tar xvpf gcc-linaro-7.5.0-2019.12-x86_64_aarch64-linux-gnu.tar.xz
```

## Build Kernel
First, install prerequisites and merge the android kernel config with the base config for raspberry pi 4 (bcm2711).

```
$ cd kernel/arpi
$ export PATH=$HOME/gcc-linaro-7.5.0-2019.12-x86_64_aarch64-linux-gnu/bin:$PATH
$ ARCH=arm64 scripts/kconfig/merge_config.sh \
  arch/arm64/configs/bcm2711_defconfig \
  kernel/configs/android-base.config \
  kernel/configs/android-recommended.config \
  ../../device/arpi/rpi4/can/linux_mcp2515.config

$ ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make Image.gz -j $(nproc)
$ ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- DTC_FLAGS="-@" make broadcom/bcm2711-rpi-4-b.dtb
$ ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- DTC_FLAGS="-@" make overlays/mcp2515-can0.dtbo
$ ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- DTC_FLAGS="-@" make overlays/vc4-kms-v3d-pi4.dtbo
$ ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- DTC_FLAGS="-@" make overlays/spi0-1cs.dtbo
$ ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- DTC_FLAGS="-@" make overlays/dwc2.dtbo
```

## Patch framework source 

To get the build to succeed we need to patch the Android framework. Change the files as described in the link below.

https://github.com/android-rpi/device_arpi_rpi4/wiki/arpi-11-:-framework-patch

## Build Android source
Continue build referring to http://source.android.com/source/building.html
```
$ source build/envsetup.sh
$ lunch rpi4car-eng
$ m ramdisk systemimage vendorimage
```

## Prepare sd card
Partitions of the card should be set-up like followings. Create a MBR partition table.

| #  | Size      | Partition | Options                            |
| -- | --------- | --------- | ---------------------------------- |
| p1 | 128MB     | boot      | set W95 FAT32(LBA) & bootable type |
| p2 | 2048MB    | /system   |                                    |
| p3 | 128MB     | /vendor   |                                    |
| p4 | remaining | /data     |                                    |
 
 > Replace mmcblkX with the name of your SD card.

 ```
$ sudo fdisk /dev/mmcblk0

Welcome to fdisk (util-linux 2.36).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Command (m for help): o

Created a new DOS disklabel with disk identifier 0x469e9837.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-124735487, default 2048): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-124735487, default 124735487): +128M

Created a new partition 1 of type 'Linux' and of size 128 MiB.

Command (m for help): p
Disk /dev/mmcblk0: 59,48 GiB, 63864569856 bytes, 124735488 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x6ee618fe


Command (m for help): n
Partition type
   p   primary (1 primary, 0 extended, 3 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (2-4, default 2): 2
First sector (264192-124735487, default 264192): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (264192-124735487, default 124735487): +2048M

Created a new partition 2 of type 'Linux' and of size 2 GiB.

Command (m for help): n
Partition type
   p   primary (2 primary, 0 extended, 2 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (3,4, default 3): 
First sector (4458496-124735487, default 4458496): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (4458496-124735487, default 124735487): +128M

Created a new partition 3 of type 'Linux' and of size 128 MiB.

Command (m for help): n
Partition type
   p   primary (3 primary, 0 extended, 1 free)
   e   extended (container for logical partitions)
Select (default e): p

Selected partition 4
First sector (4720640-124735487, default 4720640): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (4720640-124735487, default 124735487): 

Created a new partition 4 of type 'Linux' and of size 57,2 GiB.

Command (m for help): a
Partition number (1-4, default 4): 1

The bootable flag on partition 1 is enabled now.

Command (m for help): t
Partition number (1-4, default 4): 1
Hex code or alias (type L to list all): 0c

Changed type of partition 'Linux' to 'W95 FAT32 (LBA)'.

Command (m for help): p
Disk /dev/mmcblk0: 59,48 GiB, 63864569856 bytes, 124735488 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x6ee618fe

Device         Boot   Start       End   Sectors  Size Id Type
/dev/mmcblk0p1 *       2048    264191    262144  128M  c W95 FAT32 (LBA)
/dev/mmcblk0p2       264192   4458495   4194304    2G 83 Linux
/dev/mmcblk0p3      4458496   4720639    262144  128M 83 Linux
/dev/mmcblk0p4      4720640 124735487 120014848 57,2G 83 Linux

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

```
# mkfs.vfat -n boot /dev/mmcblkXp1
# mkfs.ext4 -L userdata /dev/mmcblkXp4
```

## Write system & vendor partition

 > Replace mmcblkX with the name of your SD card.
 
```
# dd if=$OUT/system.img of=/dev/mmcblkXp2 bs=1M status=progress
# dd if=$OUT/vendor.img of=/dev/mmcblkXp3 bs=1M status=progress
```
  
## Copy kernel & ramdisk to BOOT partition

```
$ cd /mount/point/of/boot
$ mkdir overlays
$ cp -a $ANDROID_BUILD_TOP/device/arpi/rpi4/boot/* .
$ cp -a $ANDROID_BUILD_TOP/kernel/arpi/arch/arm64/boot/Image.gz .
$ cp -a $ANDROID_BUILD_TOP/kernel/arpi/arch/arm64/boot/dts/broadcom/bcm2711-rpi-4-b.dtb .
$ cp -a $ANDROID_BUILD_TOP/kernel/arpi/arch/arm64/boot/dts/overlays/vc4-kms-v3d-pi4.dtbo ./overlays/
$ cp -a $ANDROID_BUILD_TOP/kernel/arpi/arch/arm64/boot/dts/overlays/spi0-1cs.dtbo ./overlays
$ cp -a $ANDROID_BUILD_TOP/kernel/arpi/arch/arm64/boot/dts/overlays/mcp2515-can0.dtbo ./overlays
$ cp -a $ANDROID_BUILD_TOP/kernel/arpi/arch/arm64/boot/dts/overlays/dwc2.dtbo ./overlays
$ cp -a $OUT/ramdisk.img .
```

## Booting the raspberry pi

Wow! You made it all the way down here, congratulations!

Now all that's left is to boot your freshly built android automotive device and start hacking away.

If you are really lucky and have a laptop that supports USB-C PD (power delivery) and a good USB-C to USB-C data cable you *might* be able to both power you device and use adb over the same cable.
This is know to work with the cable that I received with my Google Pixel 3 phone and a Lenovo Thinkpad X1 Extreme (Gen 2). Your mileage may vary.

You can also connect to the device trough adb over ethernet of wifi.

```
adb connect 127.0.0.1 # substitute for the device's ip address

# or if you have mDNS enabled

adb connect Android.local

adb shell
rpi4 #:/
```
