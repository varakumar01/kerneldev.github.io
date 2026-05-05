# Introduction:

The biggest problem I had when I initially started learning was dealing with issues at each and every step — compiling the kernel, adding/removing features, compilation errors, device-specific errors, device-specific requirements, and more. I always asked kernel-related questions and got answered with `what to do` but never `why to do it`. I've decided to make a detailed write-up that answers both the what and the why, to help new kernel developers understand and implement changes to the device tree, add NetHunter configs, and compile without errors. I'll be using the OP9 build process as a reference while keeping the terminology general enough to be broadly applicable.

## Note: If you only need to know how to compile the kernel, just skip the "Importing NetHunter Configurations" section and you'll be fine.

# Licensing the android kernel source code
The Linux kernel is licensed under the GNU General Public License v2.0. Under this license, any company that distributes a device containing the kernel — in this case, an Android smartphone — must make the complete corresponding source code of the kernel, including all modifications and configurations, available and open-source to recipients under the terms of GPL-2.0.

First, you need to find the kernel source code for your device. You can search for `<your device> kernel source`, though most vendors don't maintain their kernel with upstream changes. The simplest way to get a better kernel source is from an actively maintained custom ROM's kernel — in my case, LineageOS.

`git clone https://github.com/LineageOS/android_kernel_oneplus_sm8350.git`

Sometimes you might not find the opensource kernel for your device, then you can request your vendor to release the kernel or complain to Software Freedom Conservancy (SFC) or Free Software Foundation (FSF) Organisations as the vendor company isn't complying with the GNU General Public License v2.0 license agreement. At this point, you can either wait for compliance or abandon kernel development for this device, as it's not possible to build a working kernel without the proprietary blobs.

# Finding the Right ToolChain:

At first, finding the right toolchain might seem overwhelming, as there are many options — Neutron, TheRagingBeast, Eva, etc. — each with many different versions. I'll show you an easier way to identify the right one for your device.

There are mainly two compiler types: GCC (legacy) and Clang (current standard).

To find which toolchain is used to compile your kernel, run the following commands:

`cat /proc/version`
```
    Linux version 5.4.302-qgki-g8d4272c15d8f(runner@runnervmeorf1) (Android (13290119, +pgo, +bolt, +lto, +mlgo, based on r547379) clang version 20.0.0 (https://android.googlesource.com/toolchain/llvm-project b718bcaf8c198c82f3021447d943401e3ab5bd54), LLD 20.0.0 (/mnt/disks/build-disk/src/android/llvm-r547379-release/out/llvm-project/llvm b718bcaf8c198c82f3021447d943401e3ab5bd54)) #1 SMP PREEMPT Sat May 2 02:48:16 UTC 2026
```

`zcat /proc/config.gz | grep Compiler`
```
    #Compiler: Android (13290119, +pgo, +bolt, +lto, +mlgo, based on r547379) clang version 20.0.0 (https://android.googlesource.com/toolchain/llvm-project b718bcaf8c198c82f3021447d943401e3ab5bd54)
```

In my case it's clang version 20.0.0. You can use the same version or go slightly higher, but try to stay as close to the original version as possible — a large version gap can introduce new ABI warnings or errors that are difficult to resolve.

## Note: Before applying any NetHunter changes, make sure you can compile a working kernel from source without any modifications.

# Compiling the Kernel

Export the clang bin directory to the PATH variable to make the toolchain available:
`export PATH=/home/levion/op9/llvm-toolchain/linux-x86/clang-r547379/bin:$PATH`

Inside the Makefile, the LLVM variable decides which compiler to use.

Makefile Code Snippet:
```
ifneq ($(LLVM),)
CC		= clang
LD		= ld.lld
AR		= llvm-ar
NM		= llvm-nm
OBJCOPY		= llvm-objcopy
OBJDUMP		= llvm-objdump
READELF		= llvm-readelf
OBJSIZE		= llvm-size
STRIP		= llvm-strip
else
CC		= $(CROSS_COMPILE)gcc
LD		= $(CROSS_COMPILE)ld
AR		= $(CROSS_COMPILE)ar
NM		= $(CROSS_COMPILE)nm
OBJCOPY		= $(CROSS_COMPILE)objcopy
OBJDUMP		= $(CROSS_COMPILE)objdump
READELF		= $(CROSS_COMPILE)readelf
OBJSIZE		= $(CROSS_COMPILE)size
STRIP		= $(CROSS_COMPILE)strip
endif
```
Setting `LLVM=1` uses the Clang compiler; leaving LLVM unset (not passing it at all) falls back to GCC. All required variables are pre-defined, such as `LD = ld.lld`, etc.

To verify the paths, run the following commands:

```
which clang
which ld.lld
which llvm-ar
which llvm-nm
which llvm-objcopy
which llvm-objdump
which llvm-readelf
which llvm-size
which llvm-strip
which aarch64-linux-gnu-
which arm-linux-gnueabi-
```

This should give `/home/levion/op9/llvm-toolchain/linux-x86/clang-r547379/bin/{target}` as the result.

Since all the variables are pre-defined, the make command can be shortened to:
`make O=out ARCH=arm64 LLVM=1 CROSS_COMPILE=aarch64-linux-gnu- CROSS_COMPILE_ARM32=arm-linux-gnueabi- {target}`

However, I'll use the full explicit command throughout this write-up:
`make O=out ARCH=arm64 LLVM=1 CC=clang LD=ld.lld AR=llvm-ar NM=llvm-nm OBJCOPY=llvm-objcopy OBJDUMP=llvm-objdump READELF=llvm-readelf OBJSIZE=llvm-size STRIP=llvm-strip CROSS_COMPILE=aarch64-linux-gnu- CROSS_COMPILE_ARM32=arm-linux-gnueabi- {target}`

# Step 1: Mrproper and Clean:
Make Clean: Removes all compiled object files and other build artifacts to prepare the environment for compilation. (You can compile directly after `make clean`, provided `make defconfig` has already been run.)
`make O=out ARCH=arm64 LLVM=1 CC=clang LD=ld.lld AR=llvm-ar NM=llvm-nm OBJCOPY=llvm-objcopy OBJDUMP=llvm-objdump READELF=llvm-readelf OBJSIZE=llvm-size STRIP=llvm-strip CROSS_COMPILE=aarch64-linux-gnu- CROSS_COMPILE_ARM32=arm-linux-gnueabi- clean`

device tree
    ↓
Make defconfig (Make Clean resets to here)
    ↓
Make

Make Mrproper: Removes all object files, .config files, headers, symbol/version files, etc., giving you a completely clean build state.
`make O=out ARCH=arm64 LLVM=1 CC=clang LD=ld.lld AR=llvm-ar NM=llvm-nm OBJCOPY=llvm-objcopy OBJDUMP=llvm-objdump READELF=llvm-readelf OBJSIZE=llvm-size STRIP=llvm-strip CROSS_COMPILE=aarch64-linux-gnu- CROSS_COMPILE_ARM32=arm-linux-gnueabi- mrproper`

device tree  (Make Mrproper resets to here)
    ↓
Make defconfig
    ↓
Make

# Step 2: What is Defconfig and How to Find the Right One
Defconfig (default configuration): Predefined kernel configuration file that provides a baseline set of configuration options for building the Linux kernel for a specific architecture or device.

In simple terms, it's the minimum set of configurations needed to compile the kernel.

`.config file`: is the full set of configurations generated to compile the kernel (defconfig + all linked Kconfigs).

Folder Location: `arch/arm64/configs/` or `arch/arm64/configs/vendor`

Finding the right defconfig can be tricky, but there are a few ways to go about it:
- Fetch a working config file from the custom ROM, `adb pull /proc/config.gz` and use it as base defconfig (RELIABLE).
- Ask Custom ROM developers in Telegram groups for the defconfig they used.
- Devices sharing the same chipset will often have the same defconfig filename (though not necessarily the same config values), e.g., the Motorola Edge 30 Fusion and OnePlus 9 both use the SM8350 chipset (lahaina-qgki_defconfig).
- You could bruteforce or guess to find one.


# Make Defconfig

```
make O=out ARCH=arm64 LLVM=1 CC=clang LD=ld.lld AR=llvm-ar NM=llvm-nm OBJCOPY=llvm-objcopy OBJDUMP=llvm-objdump READELF=llvm-readelf OBJSIZE=llvm-size STRIP=llvm-strip CROSS_COMPILE=aarch64-linux-gnu- CROSS_COMPILE_ARM32=arm-linux-gnueabi- vendor/lahaina-qgki_defconfig
```

# Kernel Versioning and Backporting:
Android kernels are divided into 3 categories: Non-GKI/Pre-GKI (2.x - 4.19), GKI 1.0 (5.4), and GKI 2.0 (5.10 - 6.12).

## Tip: It's easier to backport from 6.6 → 5.10 than from 6.6 → 5.4 or 5.10 → 5.4.

- We generally don't need to reinvent the wheel - most features may have already been implemented by another developer. It's easier to follow developers and their work (commits) and understand their reasoning, for example, I've cherry-picked many commits from Madara (Sakura Kernel: 5.4) and Togo Fire (Mcquaid Kernel: 5.4) to enable qca-cld3 Inject - which allows to enable monitor mode using the internal Wi-Fi on op9 device without any external Wi-Fi adapter.

- Sometimes you'll want to cherry-pick another developer's commits. Before doing so, ensure they're using the same kernel version as yours, or you may need to backport or forward-port their work to apply it to your kernel.

- Backporting is a core skill to learn and requires much more understanding of `C`.

## Note: Skip the "Importing NetHunter Configurations" section for now — first make sure you can compile and flash a clean kernel without any errors or bootloops.

# Importing NetHunter Configurations

This is the most exciting part — importing all the necessary drivers and configurations into your tree.

Inside your device-tree:
```
git fetch https://github.com/cyberknight777/android_kernel_nethunter main
git merge -s ours --no-commit --allow-unrelated-histories --squash FETCH_HEAD
git read-tree --prefix=nethunter -u FETCH_HEAD
git commit -m "Imported nethunter/ from https://github.com/cyberknight777/android_kernel_nethunter"
```

Now, in the device-tree top-level folder, run the following:
- Open the Kconfig file and paste `source "nethunter/Kconfig"` at the end.

- Change the default to `y` inside `nethunter/Kconfig`, or enable it via menuconfig (explained below), or hardcode it in the defconfig.
```
menu "NetHunter Support"

config NETHUNTER_SUPPORT

    bool "Basic nethunter support"

    default y

```

You've almost added all the NetHunter configurations to the tree, which includes: NETHUNTER_SUPPORT, NETHUNTER_ETHERNET_SUPPORT, NETHUNTER_SDR_SUPPORT, NETHUNTER_HID_SUPPORT, NETHUNTER_USB_SUPPORT, NETHUNTER_WIFI_DRIVERS_SUPPORT, NETHUNTER_CAN_SUBSYSTEM_SUPPORT

I suggest taking an in-depth look at `nethunter/Kconfig` to understand what configurations are present and what aren't — if you need any specific ones, you can add them.

## Note: Flash `NetHunter-firmware.zip` via Magisk or KernelSU to install all the necessary firmware for external Wi-Fi adapters on your device.

# NetHunter Driver Patches:
Not all drivers are ready to use for NetHunter - most of them need patches/fixes. There are a few ways to acquire them:
- NetHunter directly provides a few patches:
    - `git clone https://gitlab.com/kalilinux/nethunter/build-scripts/kali-nethunter-kernel-builder.git` to the device-tree top-level directory.
    - Run `cd kali-nethunter-kernel-builder`, `./build.sh`
    - Select 'Apply NetHunter kernel patches' and apply the patches as per your kernel version, only accept the patches if they apply without any errors.
    - In my case, I had to apply `fix-ath9k-naming-conflict.patch` and `add-qcacld-3.0-injection-5.4.patch`
- You can acquire more patches from the NetHunter kernel developers GitHub profiles

## Note: If you are using Atheros drivers, always apply `fix-ath9k-naming-conflict.patch`.
## Tip: `fix-ath9k-naming-conflict.patch` — resolves a symbol naming conflict between ath9k and mac80211 that causes a build error.
## Tip: `add-qcacld-3.0-injection-5.4.patch` — enables monitor mode and packet injection for the internal Wi-Fi on the OP9.

# Importing Out-of-Tree Drivers (Hlcan):
In this write-up, I'll show you how to import out-of-tree drivers. It might look complicated at first, but it gets straightforward once you've done it a few times.

In the device-tree top-level folder 
`git submodule add https://github.com/V0lk3n/usb-can-2-module drivers/net/can/usb-can-2-module`
Open `drivers/net/can/Kconfig` file and paste:
`source "usb-can-2-module/Kconfig"`

Open `drivers/net/can/Makefile` file and paste:
`obj-y        += usb-can-2-module/`

Done! You've successfully imported an out-of-tree driver.

## Note: Hlcan drivers don't work alongside slcan — they conflict with each other. It's better to build both as modules (=m) instead of embedding them (=y) into the kernel.

## Tip: Try importing `https://github.com/V0lk3n/can-isotp` or any Wifi drivers which you wanna integrate into your device tree.

# Step 3: Make Menuconfig
Menuconfig is an interactive, terminal-based interface for configuring the Linux kernel. Most subsystem directories in the kernel source contain a Kconfig file that declares the options that subsystem exposes. The top-level Kconfig at the root of the source tree pulls all of them together by sourcing each one in a defined order, which determines the menu structure you see when you launch menuconfig.

There are 3 modes for each configuration: yes (y), no (n), module (m)

`make O=out ARCH=arm64 LLVM=1 CC=clang LD=ld.lld AR=llvm-ar NM=llvm-nm OBJCOPY=llvm-objcopy OBJDUMP=llvm-objdump READELF=llvm-readelf OBJSIZE=llvm-size STRIP=llvm-strip CROSS_COMPILE=aarch64-linux-gnu- CROSS_COMPILE_ARM32=arm-linux-gnueabi- menuconfig`

- If you haven't hardcoded `default y` in the Kconfig, manually enable `NetHunter Support` at the bottom of the root page and enable the hlcan driver under `Networking support → CAN bus subsystem support → CAN Device Drivers`.
- Enable any other configurations you want to enable manually.

# Step 4: Compiling the kernel
```
# Compiling the kernel (20-40mins)
make -j$(nproc) O=out ARCH=arm64 LLVM=1 CC=clang LD=ld.lld AR=llvm-ar NM=llvm-nm OBJCOPY=llvm-objcopy OBJDUMP=llvm-objdump READELF=llvm-readelf OBJSIZE=llvm-size STRIP=llvm-strip CROSS_COMPILE=aarch64-linux-gnu- CROSS_COMPILE_ARM32=arm-linux-gnueabi-

make[1]: Entering directory '/home/runner/work/Levion_kernel_OP9/Levion_kernel_OP9/out'
  GEN     Makefile
  WRAP    arch/arm64/include/generated/uapi/asm/kvm_para.h
  WRAP    arch/arm64/include/generated/uapi/asm/errno.h
  WRAP    arch/arm64/include/generated/uapi/asm/ioctl.h
  WRAP    arch/arm64/include/generated/uapi/asm/ioctls.h
  WRAP    arch/arm64/include/generated/uapi/asm/ipcbuf.h
  WRAP    arch/arm64/include/generated/uapi/asm/mman.h
  WRAP    arch/arm64/include/generated/uapi/asm/msgbuf.h
  WRAP    arch/arm64/include/generated/uapi/asm/poll.h
  .
  .
  . (After a few thousand lines of compilation)
  .
  .
  .
  LD [M]  techpack/datarmnet-ext/shs/rmnet_shs.ko
  LD [M]  techpack/datarmnet/core/rmnet_core.ko
  LD [M]  techpack/datarmnet/core/rmnet_ctl.ko
  LD [M]  techpack/display/msm/msm_drm.ko
make[1]: Leaving directory '/home/runner/work/Levion_kernel_OP9/Levion_kernel_OP9/out'
```

## Note: Errors and troubleshooting are covered in the last section.

## Note: Best practice is to always update your modules. With heavy modifications, ABI changes can prevent modules from loading — if critical modules (display, storage, etc.) fail to load, the device will freeze during boot. Even for a relatively clean build, it's still best practice to update your modules.
## Tip: If you just want to check whether your kernel boots, you can skip updating the modules for now — just flash the kernel and verify it boots first before going through all the module update steps.

# Modules Section: 
## Compiling the modules
In GKI and some Non-GKI kernels, the `make` command automatically compiles the modules.
To verify:
- Check whether there are any `*.ko` files in the out folder.
- Check the Makefile and ensure whether it compiles modules or not.

If it doesn't, compile the modules manually:
`make -j$(nproc) O=out ARCH=arm64 LLVM=1 CC=clang LD=ld.lld AR=llvm-ar NM=llvm-nm OBJCOPY=llvm-objcopy OBJDUMP=llvm-objdump READELF=llvm-readelf OBJSIZE=llvm-size STRIP=llvm-strip CROSS_COMPILE=aarch64-linux-gnu- CROSS_COMPILE_ARM32=arm-linux-gnueabi- modules`

To move all the modules to a specified folder inside the device-tree:
```
cd ./out/
make modules_install INSTALL_MOD_PATH=$PWD/modules
```

# Building vendor_dlkm.img:
## Note: If your device doesn't use the EROFS filesystem, you most likely don't need to build a `vendor_dlkm.img` — skip this step entirely.
## Tip: The following steps can look complicated. You can skip them entirely and just run `./gen_vendor_dlkm.sh` instead.

I have a OnePlus 9 running kernel version 5.4 (GKI 1.0). All GKI kernels have moved their module location from `/vendor/lib/modules/ or /system/lib/modules/ or /lib/modules/` to a dedicated modules folder `/vendor_dlkm/lib/modules/`. vendor_dlkm: Vendor Dynamic Loadable Kernel Modules.

My vendor_dlkm.img uses EROFS (Enhanced Read-Only File System). Since it's read-only, I can't directly remove or update modules — I have to rebuild the vendor_dlkm.img with the updated modules.

I have an automation script to generate vendor_dlkm.img (EROFS) — credits to @cyberknight777.

I will just explain in simple steps how to build `vendor_dlkm.img`:

- Get `vendor_dlkm.img` from your custom ROM zip
- Use the `fsck.erofs` command to extract the `vendor_dlkm.img` (from the ROM zip)
- Delete all the modules from `extracted_folder/lib/modules`
- Copy all the modules (`*.ko`) from the `./out/modules/lib` folder to the `extracted_folder/lib/modules`
- Rename `wlan.ko` to `qca_cld3_wlan.ko`, as custom ROMs expect this module name for Wi-Fi to work. (Optional)
- Debug symbols are stripped from the `.ko` modules using `llvm-objcopy --strip-debug`
- Run the depmod command against the extracted_folder which we modified and copy modules.alias, modules.dep, modules.softdep
- modules.alias, modules.dep, and modules.softdep are generated by depmod; modules.blocklist is preserved from the extracted stock image; modules.load is custom-generated.
- Fix the module paths inside modules.dep using the following commands:
`sed -i 's|[^ :]*lib/modules/[^/]*/||g' "${MODULES_DIR}/modules.dep"
sed -i 's|\([a-zA-Z0-9_.-]*\.ko\)|/vendor_dlkm/lib/modules/\1|g' "${MODULES_DIR}/modules.dep"`
- Generate modules.load:
  modules.load = all `*.ko` - EXCLUDE_SET - modules.blocklist
  EXCLUDE_SET = stock `*.ko` - modules.load - modules.blocklist (Just after extracting the stock vendor_dlkm.img)
- Rebuild the extracted folder as EROFS using the `mkfs.erofs` command

## Tip: You can update the fstab (fstab.default, recovery.fstab) to enable support for both ext4 and EROFS for vendor_dlkm, making it easier to update modules in the future.

fstab Code Snippet:

```
#<src>                                                 <mnt_point>            <type>  <mnt_flags and options>                            <fs_mgr_flags>
system                                                  /system                ext4    ro,barrier=1,discard                                 wait,slotselect,avb=vbmeta_system,logical,first_stage_mount,avb_keys=/avb/q-gsi.avbpubkey:/avb/r-gsi.avbpubkey:/avb/s-gsi.avbpubkey
system_ext                                              /system_ext            ext4    ro,barrier=1,discard                                 wait,slotselect,avb=vbmeta_system,logical,first_stage_mount
product                                                 /product               ext4    ro,barrier=1,discard                                 wait,slotselect,avb=vbmeta_system,logical,first_stage_mount
vendor                                                  /vendor                erofs   ro                                                   wait,slotselect,avb=vbmeta_vendor,logical,first_stage_mount
vendor                                                  /vendor                ext4    ro,barrier=1,discard                                 wait,slotselect,avb=vbmeta_vendor,logical,first_stage_mount
vendor_dlkm                                             /vendor_dlkm           erofs   ro                                                   wait,slotselect,logical,first_stage_mount
vendor_dlkm                                             /vendor_dlkm           ext4    ro,barrier=1,discard                                 wait,slotselect,logical,first_stage_mount
odm                                                     /odm                   erofs   ro                                                   wait,slotselect,avb=vbmeta_vendor,logical,first_stage_mount
odm                                                     /odm                   ext4    ro,barrier=1,discard                                 wait,slotselect,avb=vbmeta_vendor,logical,first_stage_mount
/dev/block/by-name/metadata                             /metadata              ext4    noatime,nosuid,nodev,discard                         wait,check,formattable,first_stage_mount
/dev/block/bootdevice/by-name/persist                   /mnt/vendor/persist    ext4    noatime,nosuid,nodev,barrier=1                       wait
/dev/block/bootdevice/by-name/userdata                  /data                  f2fs    noatime,nosuid,nodev,discard,inlinecrypt,reserve_root=32768,resgid=1065,fsync_mode=nobarrier    latemount,wait,check,formattable,fileencryption=aes-256-xts:aes-256-cts:v2+inlinecrypt_optimized+wrappedkey_v0,keydirectory=/metadata/vold/metadata_encryption,metadata_encryption=aes-256-xts:wrappedkey_v0,quota,reservedsize=128M,sysfs_path=/sys/devices/platform/soc/1d84000.ufshc,checkpoint=fs
/dev/block/bootdevice/by-name/misc                      /misc                  emmc    defaults                                             defaults
/devices/platform/soc/8804000.sdhci/mmc_host*           /storage/sdcard1       vfat    nosuid,nodev                                         wait,voldmanaged=sdcard1:auto,encryptable=footer
/devices/platform/soc/1da4000.ufshc_card/host*          /storage/sdcard1       vfat    nosuid,nodev                                         wait,voldmanaged=sdcard1:auto,encryptable=footer
/devices/platform/soc/*.ssusb/*.dwc3/xhci-hcd.*.auto*   /storage/usbotg        vfat    nosuid,nodev                                         wait,voldmanaged=usbotg:auto
/dev/block/bootdevice/by-name/modem                     /vendor/firmware_mnt   vfat    ro,shortname=lower,uid=1000,gid=1000,dmask=227,fmask=337,context=u:object_r:firmware_file:s0 wait,slotselect
/dev/block/bootdevice/by-name/dsp                       /vendor/dsp            ext4    ro,nosuid,nodev,barrier=1                            wait,slotselect
/dev/block/bootdevice/by-name/bluetooth                 /vendor/bt_firmware    vfat    ro,shortname=lower,uid=1002,gid=3002,dmask=227,fmask=337,context=u:object_r:bt_firmware_file:s0 wait,slotselect
/dev/block/bootdevice/by-name/qmcs                      /mnt/vendor/qmcs       vfat    noatime,nosuid,nodev,context=u:object_r:vendor_qmcs_file:s0   wait,check,formattable
```

# Updating the Modules:
## Tip: You don't have to update the modules if you embed everything directly into the kernel (=y).
Let's go with actually updating the modules:
- Manual Method (Root Required): Copy all modules (`*.ko`) from `device-tree/out/modules/lib/modules` to your device's module folder `/vendor/lib/modules/ or /system/lib/modules/ or /lib/modules/ or /vendor_dlkm/lib/modules/`
- Automated Method: AnyKernel3 allows you to update modules. Copy all modules (`*.ko`) from `device-tree/out/modules/lib/modules` to the `ak3/modules/system/lib/modules` folder; the remaining steps are covered in the AnyKernel3 section.
  - You need to maintain the same folder structure — for example, if your device uses `/lib/modules/`, the AK3 path should also be `ak3/lib/modules/`, and vice versa.

## Tip: Update the vendor_ramdisk modules if available — for the OP9, this was mandatory from Android 16 ROMs onwards.


# Creating an Anykernel3 zip file

Anykernel3 (AK3): AnyKernel3 provides a controlled and modular mechanism to update or replace kernel binaries, kernel modules, boot images, and apply low-level system patches, while maintaining compatibility with the device’s current firmware and partition layout. It also offers granular control over the installation process, allowing selective modification—so specific components can be upgraded or left untouched as required.

## Note: Read AK3's README.md for detailed instructions and commands.
## Tip: AK3 has many template variants, ranging from simple to complex. I'll try to keep things as straightforward as possible while explaining my thought process.

- Cloning the repo: `git clone https://github.com/osm0sis/AnyKernel3.git`
- In AK3, we maintain the same folder structure as what we want to modify. In my case, I wanted to modify `fstab.default, recovery.fstab` (to add ext4 support), and the vendor ramdisk modules, so I extracted `vendor_boot.img` using these commands:
```
unpack_bootimg --boot_img vendor_boot.img --out out_vendor_boot
cd out_vendor_boot
file vendor_ramdisk
lz4 -d vendor_ramdisk vendor_ramdisk.cpio
mkdir ramdisk
cd ramdisk
cpio -idmv < ../vendor_ramdisk.cpio
```
After observing the folder structure:
  - fstab.default is in `out_vendor_boot/ramdisk/first_stage_ramdisk/system/etc`
  - recovery.fstab is in `out_vendor_boot/ramdisk/system/etc`
  - vendor ramdisk modules are present in `out_vendor_boot/ramdisk/lib/modules`
  - As per AK3's README.md, you can use either `ramdisk` or `vendor_ramdisk` as the folder name.

Anykernel3 Folder Structure based on my vendor_boot.img (`tree -h`):
```
└── [4.0K]  vendor_ramdisk
    ├── [4.0K]  first_stage_ramdisk
    │   └── [4.0K]  system
    │       └── [4.0K]  etc
    │           └── [6.0K]  fstab.default
    ├── [4.0K]  lib
    │   └── [4.0K]  modules
    │       ├── [ 27K]  adsp_loader_dlkm.ko
    │       ├── [ 67K]  apr_dlkm.ko
    │       ├── [4.9M]  msm_drm.ko
    │       ├── [ 34K]  q6_notifier_dlkm.ko
    │       ├── [ 19K]  q6_pdr_dlkm.ko
    │       └── [ 24K]  snd_event_dlkm.ko
    └── [4.0K]  system
        └── [4.0K]  etc
            └── [6.0K]  recovery.fstab
```

- Any `.img` file placed inside the AK3 folder will be flashed automatically, provided the filename is correct (e.g., `vendor_dlkm.img, dtb.img, dtbo.img, vbmeta.img`). In my case, I include `vendor_dlkm.img`, which we generated earlier.
- We need to copy `Image(Image, Image.gz, Image.gz-dtb, Image-dtb)`, `dtb.img, dtbo.img`(optional if compiled) file from `device-tree/out/arch/arm64/boot` to `ak3/Image`
- The main configuration for AK3 is the `anykernel.sh` script file. Here are the most important variables:
  - `do.modules=1`: if you want AK3 to copy all the modules automatically to the device.
  - `do.modules=0`: You can do it manually using the `cp` command from the modules folder.
  - `do.systemless=1` (used with `do.modules=1`): AK3 creates a Magisk/KernelSU helper module that overlays the modules. If the kernel changes, the helper module automatically removes itself to prevent conflicts.
  - `do.systemless=0` (used with `do.modules=1`): Modules are directly updated/replaced. Recommended for NetHunter builds.
  - `device.name*`: Get the device names from the `build.prop` file, or just type them out.
  - `supported.versions=16`,`supported.versions=15-16`: Supported Android versions.
  - `BLOCK`: Decides which partition to target (`BLOCK=boot, BLOCK=vendor_boot`)
  - `IS_SLOT_DEVICE=1`: set this if your device has A/B partitions. The easiest way to check: `ls /dev/block/by-name | grep -E 'boot_a|boot_b'` or `getprop ro.build.ab_update`
  - `dump_boot` (from tools/ak3-core.sh): dumps and splits the image, then extracts the ramdisk.
  - `write_boot` (from tools/ak3-core.sh): repacks the ramdisk, then builds, signs, and writes the image along with vendor_dlkm and dtbo.

## Tip: You can include extra binaries in the tools folder and call them from anykernel.sh to perform custom actions.
## Tip: The easiest way to get a properly working AK3 setup is to find an existing custom kernel for your device and use its AK3 as a base template.

# Building boot.img using Boot Editor (Optional)
## Note: This method cannot update modules and is less flexible than AnyKernel3.

- Download and unzip the latest release from this repo: https://github.com/cfig/Android_boot_image_editor/releases
- Copy the latest working boot.img from your custom ROM to the boot-editor folder.
- Run `./gradlew unpack`
- Copy and rename the Image file from the `device-tree/out/arch/arm64/boot` folder to `boot-editor/build/unzip_boot/kernel`
- Run `./gradlew pack` from the boot-editor root folder
- Output: `boot.img.signed`

# Errors Section:
I can't solve every problem in this section, but I can guide you through solving them yourself.
- Most errors don't actually require an extensive knowledge of C to resolve:
    - Disable the CONFIG_CC_WERROR config option, which promotes higher-severity warnings to errors.
    - You may have chosen the wrong toolchain — try using the exact toolchain identified from your device info. (`cat /proc/version` or `zcat /proc/config.gz | grep Compiler`)
    - Check whether the Android kernel source is well-maintained, or use the exact one your device's custom ROM uses.
    - Ensure all exported variables — clang, ld.lld, llvm-ar, llvm-nm, etc. — point to the same toolchain, or use the full explicit command like I do.
    - Ensure you're using the correct defconfig. If unsure, use `/proc/config.gz` as your base — it's very reliable.
- If errors still persist, you'll need to dig into them manually, which requires a stronger foundation in `C`.

# GitHub Profiles:
Here are a few GitHub profiles I've mentioned — they can help with cherry-picking new features or resolving errors.
- NetHunter: @cyberknight777, @MrR0b0X, @kimocoder(Aircrack-ng), @V0lk3n (CAN)
- Kernel Backports: @backslashxx, @sidex15
- General (NH + kerneldev): @Madara273, @TogoFire