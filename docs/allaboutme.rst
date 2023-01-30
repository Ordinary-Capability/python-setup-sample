############
All about me
############

## fy12b image build
Compile and build bootloader images.
### Bootloader
```
make CROSS_COMPILE=aarch64-fullhan-linux-gnu-   PLAT=fullhan SPD=opteed BL32=/home/FULLHAN/zhengk315/fy12b/tee-pager_v2.bin  BL33=/home/FULLHAN/zhengk315/fy12b/u-boot-dtb.bin  all fip

./build.sh CROSS_COMPILE=aarch64-fullhan-linux-gnu- BL32=/data1/zhengk315/fy12/tee-pager_v2.bin  BL33=/data1/zhengk315/fy12/u-boot-dtb2.bin  all fip

make ARCH=arm CROSS_COMPILE=aarch64-fullhan-linux-gnu- molchip_uboot_defconfig

../uboot/tools/mkmolchipboot 0x30800000 bl1.bin bl1.img

 cat u-boot-spl-header.img bl1.img bl1.img > bl1-with-spl.img

dd if=u-boot-spl-header.img of=atf-tee-spl.img  bs=64k
dd if=bl1.img of=atf-tee-spl.img  bs=64k seek=1

dd if=bl1-with-spl.img of=atf-tee-spl.img
dd if=fip.bin of=atf-tee-spl.img  bs=64k seek=2

dd if=u-boot-spl-header.img of=bl1-with-spl.img  bs=64k
dd if=bl1.img of=bl1-with-spl.img  bs=64k seek=1
```
### Boot images
Make boot and vendor boot images and add hash footers.
```
mkbootimg.py --header_version 4 --kernel Image --dtb molchip-kernel.dtb --ramdisk ramdisk.img  --os_version 5.10.109 --os_patch_level 2022-11-08 --board FY12B -o boot.img --vendor_boot vendor_boot.img

/home/FULLHAN/zhengk315/aosp/system/tools/mkbootimg/mkbootimg.py \
        --header_version 4 --kernel Image --dtb molchip-kernel.dtb \ 
        --ramdisk ramdisk.img  --os_version 5.10.109 --os_patch_level 2022-11-08 \
        --board FY12B -o boot.img --vendor_boot vendor_boot.img

/home/FULLHAN/zhengk315/aosp/external/avb/avbtool.py add_hash_footer \
        --image boot.img --partition_size 52428800 	--partition_name boot \
        --key testkey_rsa4096.pem --algorithm SHA256_RSA4096

/home/FULLHAN/zhengk315/aosp/external/avb/avbtool.py add_hash_footer \
        --image vendor_boot.img --partition_size 11534336 --partition_name vendor_boot \
        --key testkey_rsa4096.pem --algorithm SHA256_RSA4096
```
### System and Vender images
Make verdor image, then add hashtree footer to system and vendor images.
```
/home/FULLHAN/zhengk315/aosp/out/host/linux-x86/bin/build_image vendor vendor_image_info.txt vendor.img
/home/FULLHAN/zhengk315/aosp/external/avb/avbtool.py add_hashtree_footer \
    --image system.img --partition_size 3029336064 --partition_name system \
    --key testkey_rsa4096.pem --algorithm SHA256_RSA4096 --do_not_generate_fec

/home/FULLHAN/zhengk315/aosp/external/avb/avbtool.py add_hashtree_footer \
        --image vendor.img --partition_size 524288000 --partition_name vendor \
        --key testkey_rsa4096.pem --algorithm SHA256_RSA4096 --do_not_generate_fec
```
### Userdata image
Make userdata image and add its hash footer.
```
dd if=/dev/zero of=userdata.img bs=1M count=600
mkfs.ext4 userdata.img

/home/FULLHAN/zhengk315/aosp/external/avb/avbtool.py add_hash_footer \
        --image userdata.img --partition_size 668991488 --partition_name userdata \
        --key testkey_rsa4096.pem --algorithm SHA256_RSA4096
```
### Vbmeta image
Make vbmeta image using images built previously.
```
/home/FULLHAN/zhengk315/aosp/external/avb/avbtool.py make_vbmeta_image \
        --output vbmeta.img --key testkey_rsa4096.pem --algorithm SHA256_RSA4096 \
	    --include_descriptors_from_image boot.img \
	    --include_descriptors_from_image vendor_boot.img \
	    --include_descriptors_from_image system.img \
	    --include_descriptors_from_image vendor.img
```

### Burn images
Burn images via tftp or **dd** command.
```
tftpboot 0x150000000 fy12b/emmc_user_data.img
mmc write 0x150000000 0 0x806

tftpboot 0x150000000 fy12b/vbmeta.img
tftpboot 0x150000000 fy12b/vbmeta.new.img
mmc write 0x150000000 0x800 0x6

tftpboot 0x150000000 fy12b/boot.img
tftpboot 0x150000000 fy12b/boot.new.img
mmc write 0x150000000 0x1000 0x19000

tftpboot 0x150000000 fy12b/vendor_boot.img
mmc write 0x150000000 0x1a000 0x5800

tftpboot 0x150000000 fy12b/vendor.img
tftpboot 0x150000000 fy12b/vendor.old.img
mmc write 0x150000000 0x5c4000 0xfa000

tftpboot 0x150000000 fy12b/userdata.img
mmc write 0x150000000 0x6be000 0x13f000

tftpboot 0x140080000 fy12b/ap_kernel.img
tftpboot 0x142000000 fy12b/ap.dtb
booti 0x140080000 - 0x142000000
ifconfig eth0 192.168.72.131
mount -t nfs -o nolock 192.168.72.20:/smb/fy12b /mnt
mount -t nfs -o nolock,addr=192.168.72.20 192.168.72.20:/smb/fy12b /mnt/nfs

dd if=system.img of=/dev/mmcblk0p4
dd if=vendor.img of=/dev/mmcblk0p5
dd if=userdata.img of=/dev/mmcblk0p6
```
### Other commands
```
/home/FULLHAN/zhengk315/aosp/out/host/linux-x86/bin/build_image vendor vendor_image_info.txt vendor.img .

mount -o rw,remount /vendor
mount -o rw,remount /

sudo mount -o ro system_sparse.img tmp2

build/make/target/board/BoardConfigMainlineCommon.mk
```

