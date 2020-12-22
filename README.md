# Android Automotive Raspberry Pi 4B

## Hardware

* Raspberry Pi 4B
* 7" HDMI+USB Touchscreen
* PiCAN2 hat
* OBD-II to D-SUB adapter

Start by reading : [https://github.com/PontusPersson/local_manifests](https://github.com/PontusPersson/local_manifests)

## ARM64 Toolchain

> TODO

## Build Kernel
First, install prerequisites and merge the android kernel config with the base config for raspberry pi 4 (bcm2711).

```
$ sudo apt install gcc-aarch64-linux-gnu libssl-dev
$ cd kernel/arpi
$ ARCH=arm64 scripts/kconfig/merge_config.sh \
  arch/arm64/configs/bcm2711_defconfig \
  kernel/configs/android-base.config \
  kernel/configs/android-recommended.config \
  ../../device/arpi/rpi4/can/linux_mcp2515.config

$ ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make Image.gz -j $(nproc)
$ ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- DTC_FLAGS="-@" make broadcom/bcm2711-rpi-4-b.dtb
$ ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- DTC_FLAGS="-@" make overlays/vc4-kms-v3d-pi4.dtbo
$ ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- DTC_FLAGS="-@" make overlays/spi0-1cs.dtbo
$ ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- DTC_FLAGS="-@" make overlays/dwc2.dtbo
```

## Patch framework source 

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
$ cp -a $ANDROID_BUILD_TOP/kernel/arpi/arch/arm64/boot/bcm2711-rpi-4-b.dtb .
$ cp -a $ANDROID_BUILD_TOP/kernel/arpi/arch/arm64/boot/vc4-kms-v3d-pi4.dtbo ./overlays/
$ cp -a $ANDROID_BUILD_TOP/kernel/arpi/arch/arm64/boot/spi0-1cs.dtbo ./overlays
$ cp -a $ANDROID_BUILD_TOP/kernel/arpi/arch/arm64/boot/dwc2.dtbo ./overlays
$ cp -a $OUT/ramdisk.img .
```