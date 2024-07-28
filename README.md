# Synology DSM 7 - Compiling DVB Linux Kernel Modules
I'm writing this since it could be helpful to everyone trying to make common USB DVB work on Synology DSM 7 and above.

Some specs:

- Model: Synology DS1821+ 
- Arch: x86_64
- Core Name: v1000
- OS: DSM 7.2.1-69057 Update 5
- Linux Kernel 4.4.302+
- The PCI-DVB device I'm using is a DigitalDevices Cine C2 PCIe card 
  - This device is reported to be supported inside the Linux Kernel from 4.17, but the Synology Kernel is 4.4.302+. So, we'll have to compile the kernel modules ourselves.

This guide will use the `media_build` repo from [**Linuxtv**](https://git.linuxtv.org/media_build.git) to compile the kernel modules. This repo backports patches to use more recent devices in legacy kernels. This repo is EOL, but it should work for a myriad of devices, and there are no other alternatives to my knowledge.


## Acknowledgements
First of all, I want to thank [**@th0ma7**](https://github.com/th0ma7) for the work on [**his Synology repo**](https://github.com/th0ma7/synology). His guide on how to compile kernel modules for Synology NAS (DSM 6) was very helpful to me, and I wanted to port it to DSM 7 since I could not find any guide for it.

Then, I want to thank [**@b-rad-NDi**](https://github.com/b-rad-NDi) for the work on the [**Embedded-MediaDrivers**](https://github.com/b-rad-NDi/Embedded-MediaDrivers) repo.

They both made my life so much easier for compiling and installing the kernel modules with the Synology toolchain. I used their work as a base for this guide.

## Warning
I cannot, and will not, be held responsible for any damage to your Synology. 

Moreover, this guide is provided as-is: I can't guarantee that it will work for other architectures, in fact it's known that **there are** architectures that have problems with compiling the kernel modules.

I'm just sharing what worked for me and my v1000 architecture.

## Prerequisites
- A Synology NAS with DSM 7 installed
- A USB DVB device supported by the Linux Kernel (check [**this list**](https://www.linuxtv.org/wiki/index.php/DVB-T_USB_Devices) for reference)
- The USB Serial drivers already installed and loaded (follow [**this guide**](https://mariushosting.com/synology-how-to-add-usb-support-on-dsm-7/) if you need help)

# Step 1: Prepare the build environment
First of all, we need to prepare the build environment. 
I chose to do everything in a Docker container, to keep my system clean and tidy.

So I spun up an Ubuntu 22.04 container:

```bash
docker run -it --platform linux/amd64 --name dsm7-dvb-build -v <folder>:/export ubuntu:latest bash
```
   
   where `<folder>` is the path to your actual system you'll use as a bridge between the container and your system.

Now update and install the required packages:

```bash
apt update
apt install build-essential ncurses-dev bc libssl-dev libc6-i386 curl libproc-processtable-perl git wget kmod
```

Then, create a folder with all the tools needed for the compilation:
   
```bash
mkdir /compile
cd /compile
git clone https://github.com/b-rad-NDi/Embedded-MediaDrivers
```

This will clone the Embedded-MediaDrivers repo, which contains the tool you'll use to compile the kernel modules.

# Step 2: Download the toolchain and the GPL sources
## Synology Toolchain
On DSM 6 you could find the toolchain [**here**](https://sourceforge.net/projects/dsgpl/files/), but on DSM 7 and later they have been moved to: https://archive.synology.com/download/ToolChain.

So you'll have to browse the folders and download the appropriate version for your OS and your architecture. Once downloaded, transfer them to your container and put them in (create the folder, change for your architecture):
```bash
/compile/Embedded-MediaDrivers/dl/SYNO-V1000
```
- You can also use the wget in the container to download the files directly in the right folder.

## GPL Sources
Browse again the [archive.synology.com/download/ToolChain](https://archive.synology.com/download/ToolChain) folder and download the Synology NAS GPL sources for your architecture. Download the `linux-<kernelversion>.tgz` (in my case `linux-4.4.x.txz`) and transfer it to your container, in:
```bash
/compile/Embedded-MediaDrivers/dl/SYNO-V1000
```
- The same wget shortcut applies here too.


# Step 3: Configure the tool
## Creating the config file
Enter the folder, and in the `config` subfolder, you'll have to create a file containing the config for your architecture. To create mine, I used the b-rad-NDi one as a base (SYNO-Apollolake.conf) and modified for my v1000 architecture (SYNO-V1000.conf). In particular, I did: 

- find and replace everything from Apollo to Gemini (make sure to replace always with the correct case).
- Adjust the TOOLCHAIN_LOCATION var at the top of the file to match the Toolchain file name you've downloaded before.
- In my case, the KERNEL_LOCATION var was already correct, but you should check it too.
- Adjust the dead Linuxtv media_build repo link (copy mine)

You can find my modified file in this repo, in the `config` folder.


## Creating the board folder
Enter the Embedded-MediaDrivers folder and create a new subfolder in the `board` subfolder (mine is SYNO-V1000). This folder contains the board-specific patches for the kernel modules. 

b-rad-NDi already created one for Apollo Lake, so I just copied it and modified it for v1000. In his repo there are also other boards, so you can use them as a base for your architecture. On x86_64, it should be pretty straightforward to modify the Apollo Lake one for your architecture. For other architectures, you'll have to do some research.

- You can find my modified folder in this repo, in the `board` folder.

## Initializing the tool
I ran:
```bash
./md_builder.sh -i -d SYNO-V1000
```
to initialize the tool. This will create a `SYNO-V1000` folder in the `build` folder, containing the extracted kernel sources and toolchain.

# Step 4: Build the Synology Linux kernel
## Compile the headers for the downloaded kernel
I ran:
```bash
./md_builder.sh -B media -d SYNO-V1000
```
to compile the Synology kernel. This took a bit. If you want to speed up the process, you can edit the `config` file to leverage make multi-threading, but if you're not familiar with it, I suggest you don't and just wait.

## Manipulating the media_build repo
This step is not required with the for SYNO-V1000.conf file in this repo.
in the `build` folder, you'll have now a new folder called `media_build` containing the Linuxtv repo. Go to this folder. Since this repo is EOL and files were deleted, I checked out the last working commit:
```bash
git checkout 0fe857b86addf382f6fd383948bd7736a3201403
```

Then, I opened the file `build` and commented out the lines that made the tool check for the latest version (the one that deletes the files). In particular, lines 504-505:

```bash
	print "****************************\n";
	print "Updating the building system\n";
	print "****************************\n";
	#run("git pull git://linuxtv.org/media_build.git master",
	#    "Can't clone tree from linuxtv.org");

	run("make -C linux/ download", "Download failed");
	run("make -C linux/ untar", "Untar failed");
```

### Extra (Only for v1000?)
I had to remove a specific patch in the `media_build/backports` folder, since it was causing the compilation to fail. The patch is `v4.11_vb2_kmap.patch`. 

This patch is just wrong for lots of kernels and has been reverted since (but not in this repo). Don't delete the file, just empty it.

# Step 5: Compiling the DVB kernel modules
Now, everything is ready to run the actual compilation:
```bash
./md_builder.sh -B media -d SYNO-V1000
```

This will take a while. Just like before, you can speed up the process by editing the makefiles to leverage make multi-threading.

If everything goes well you'll find the compiled kernel modules in the `build/media_build/v4l` folder. You'll need only the `.ko` files. 

I couldn't compile the kernel modules. Below is the terminal log.

```
root@07acd4a885d4:/compile/Embedded-MediaDrivers# ./md_builder.sh -g -d SYNO-V1000
configure
Device: SYNO-V1000
custom environment
custom kernel configure func
scripts/kconfig/conf  --olddefconfig Kconfig
#
# configuration written to .config
#
custom media configure func
make -C /compile/Embedded-MediaDrivers/build/SYNOV1000/media_build/v4l release
make[1]: Entering directory '/compile/Embedded-MediaDrivers/build/SYNOV1000/media_build/v4l'
Searching in /compile/Embedded-MediaDrivers/build/SYNOV1000/linux-4.4.x/Makefile for kernel version.
Forcing compiling to version 4.4.302
make[1]: Leaving directory '/compile/Embedded-MediaDrivers/build/SYNOV1000/media_build/v4l'
make: Entering directory '/compile/Embedded-MediaDrivers/build/SYNOV1000/media_build/linux'
wget http://linuxtv.org/downloads/drivers/linux-media-LATEST.tar.bz2.md5 -O linux-media.tar.bz2.md5.tmp
URL transformed to HTTPS due to an HSTS policy
--2024-07-28 15:39:41--  https://linuxtv.org/downloads/drivers/linux-media-LATEST.tar.bz2.md5
Resolving linuxtv.org (linuxtv.org)... 140.211.166.241
Connecting to linuxtv.org (linuxtv.org)|140.211.166.241|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 105 [application/x-bzip2]
Saving to: 'linux-media.tar.bz2.md5.tmp'

linux-media.tar.bz2.md5.tmp                  100%[==============================================================================================>]     105  --.-KB/s    in 0s

2024-07-28 15:39:41 (77.0 MB/s) - 'linux-media.tar.bz2.md5.tmp' saved [105/105]

cat: linux-media.tar.bz2.md5: No such file or directory
URL transformed to HTTPS due to an HSTS policy
--2024-07-28 15:39:41--  https://linuxtv.org/downloads/drivers/linux-media-LATEST.tar.bz2
Resolving linuxtv.org (linuxtv.org)... 140.211.166.241
Connecting to linuxtv.org (linuxtv.org)|140.211.166.241|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 7528232 (7.2M) [application/x-bzip2]
Saving to: 'linux-media.tar.bz2'

linux-media.tar.bz2                          100%[==============================================================================================>]   7.18M  4.27MB/s    in 1.7s

2024-07-28 15:39:44 (4.27 MB/s) - 'linux-media.tar.bz2' saved [7528232/7528232]

make: Leaving directory '/compile/Embedded-MediaDrivers/build/SYNOV1000/media_build/linux'
make: Entering directory '/compile/Embedded-MediaDrivers/build/SYNOV1000/media_build/linux'
tar xfj linux-media.tar.bz2
rm -f .patches_applied .linked_dir .git_log.md5
make: Leaving directory '/compile/Embedded-MediaDrivers/build/SYNOV1000/media_build/linux'
root@07acd4a885d4:/compile/Embedded-MediaDrivers# ./md_builder.sh -B media -d SYNO-V1000
build media
Device: SYNO-V1000
custom environment
custom kernel build func
scripts/kconfig/conf  --silentoldconfig Kconfig
  CHK     include/config/kernel.release
  CHK     include/generated/uapi/linux/version.h
  CHK     include/generated/utsrelease.h
  CHK     include/generated/bounds.h
  CHK     include/generated/timeconst.h
  CHK     include/generated/asm-offsets.h
  CALL    scripts/checksyscalls.sh
  CHK     scripts/mod/devicetable-offsets.h
  Building modules, stage 2.
  MODPOST 517 modules
custom media build func
make: Entering directory '/compile/Embedded-MediaDrivers/build/SYNOV1000/media_build/v4l'
Updating/Creating .config
make[1]: Entering directory '/compile/Embedded-MediaDrivers/build/SYNOV1000/media_build/linux'
Applying patches for kernel 4.4.302
patch -s -f -N -p1 -i ../backports/api_version.patch
patch -s -f -N -p1 -i ../backports/pr_fmt.patch
patch -s -f -N -p1 -i ../backports/debug.patch
patch -s -f -N -p1 -i ../backports/drx39xxj.patch
patch -s -f -N -p1 -i ../backports/ccs.patch
patch -s -f -N -p1 -i ../backports/rc-cec.patch
patch -s -f -N -p1 -i ../backports/v5.17_spi.patch
patch -s -f -N -p1 -i ../backports/v5.17_iosys.patch
patch -s -f -N -p1 -i ../backports/v5.17_overflow.patch
patch -s -f -N -p1 -i ../backports/v5.15_container_of.patch
patch -s -f -N -p1 -i ../backports/v5.14_bus_void_return.patch
patch -s -f -N -p1 -i ../backports/v5.13_atmel.patch
patch -s -f -N -p1 -i ../backports/v5.13_stk1160.patch
patch -s -f -N -p1 -i ../backports/v5.12_uvc.patch
patch -s -f -N -p1 -i ../backports/v5.11_isa.patch
patch -s -f -N -p1 -i ../backports/v5.10_vb2_dma_buf_map.patch
patch -s -f -N -p1 -i ../backports/v5.9_tasklet.patch
patch -s -f -N -p1 -i ../backports/v5.9_netup_unidvb_devm_revert.patch
patch -s -f -N -p1 -i ../backports/v5.7_mmap_read_lock.patch
patch -s -f -N -p1 -i ../backports/v5.7_vm_map_ram.patch
patch -s -f -N -p1 -i ../backports/v5.7_pin_user_pages.patch
patch -s -f -N -p1 -i ../backports/v5.7_define_seq_attribute.patch
patch -s -f -N -p1 -i ../backports/v5.6_pin_user_pages.patch
patch -s -f -N -p1 -i ../backports/v5.6_const_fb_ops.patch
patch -s -f -N -p1 -i ../backports/v5.6_pm_runtime_get_if_active.patch
patch -s -f -N -p1 -i ../backports/v5.5_alsa_pcm_api_updates.patch
patch -s -f -N -p1 -i ../backports/v5.5_memtype_h.patch
patch -s -f -N -p1 -i ../backports/v5.5_dev_printk_h.patch
patch -s -f -N -p1 -i ../backports/v5.5_vb2_kmap.patch
patch -s -f -N -p1 -i ../backports/v5.5_go7007.patch
patch -s -f -N -p1 -i ../backports/v5.4_revert_spi_transfer.patch
patch -s -f -N -p1 -i ../backports/v5.4_async.patch
patch -s -f -N -p1 -i ../backports/v5.1_vm_map_pages.patch
patch -s -f -N -p1 -i ../backports/v5.1_devm_i2c_new_dummy_device.patch
patch -s -f -N -p1 -i ../backports/v5.0_ipu3-cio2.patch
patch -s -f -N -p1 -i ../backports/v5.0_time32.patch
patch -s -f -N -p1 -i ../backports/v5.0_gpio.patch
patch -s -f -N -p1 -i ../backports/v4.20_access_ok.patch
patch -s -f -N -p1 -i ../backports/v4.18_fwnode_args_args.patch
patch -s -f -N -p1 -i ../backports/v4.18_ccs_bitops.patch
patch -s -f -N -p1 -i ../backports/v4.18_vb2_map_atomic.patch
patch -s -f -N -p1 -i ../backports/v4.17_i2c_check_num_msgs.patch
patch -s -f -N -p1 -i ../backports/v4.16_poll_requested_events.patch
patch -s -f -N -p1 -i ../backports/v4.15_pmdown_time.patch
patch -s -f -N -p1 -i ../backports/v4.15_async.patch
patch -s -f -N -p1 -i ../backports/v4.14_saa7146_timer_cast.patch
patch -s -f -N -p1 -i ../backports/v4.14_module_param_call.patch
patch -s -f -N -p1 -i ../backports/v4.14_fwnode_handle_get.patch
patch -s -f -N -p1 -i ../backports/v4.13_remove_nospec_h.patch
patch -s -f -N -p1 -i ../backports/v4.13_drmP.patch
patch -s -f -N -p1 -i ../backports/v4.13_fwnode_graph_get_port_parent.patch
patch -s -f -N -p1 -i ../backports/v4.12_revert_solo6x10_copykerneluser.patch
patch -s -f -N -p1 -i ../backports/v4.11_drop_drm_file.patch
patch -s -f -N -p1 -i ../backports/v4.11_vb2_kmap.patch
patch -s -f -N -p1 -i ../backports/v4.11_pwc.patch
patch -s -f -N -p1 -i ../backports/v4.10_sched_signal.patch
patch -s -f -N -p1 -i ../backports/v4.10_fault_page.patch
patch -s -f -N -p1 -i ../backports/v4.10_refcount.patch
patch -s -f -N -p1 -i ../backports/v4.9_mm_address.patch
patch -s -f -N -p1 -i ../backports/v4.9_dvb_net_max_mtu.patch
patch -s -f -N -p1 -i ../backports/v4.9_probe_new.patch
patch -s -f -N -p1 -i ../backports/v4.9_vivid_ktime.patch
patch -s -f -N -p1 -i ../backports/v4.8_user_pages_flag.patch
patch -s -f -N -p1 -i ../backports/v4.8_em28xx_bitfield.patch
patch -s -f -N -p1 -i ../backports/v4.8_dma_map_resource.patch
patch -s -f -N -p1 -i ../backports/v4.8_drm_crtc.patch
patch -s -f -N -p1 -i ../backports/v4.7_dma_attrs.patch
patch -s -f -N -p1 -i ../backports/v4.7_pci_alloc_irq_vectors.patch
patch -s -f -N -p1 -i ../backports/v4.7_copy_to_user_warning.patch
patch -s -f -N -p1 -i ../backports/v4.7_objtool_warning.patch
patch -s -f -N -p1 -i ../backports/v4.6_i2c_mux.patch
patch -s -f -N -p1 -i ../backports/v4.5_gpiochip_data_pointer.patch
patch -s -f -N -p1 -i ../backports/v4.5_get_user_pages.patch
patch -s -f -N -p1 -i ../backports/v4.5_uvc_super_plus.patch
patch -s -f -N -p1 -i ../backports/v4.5_copy_to_user_warning.patch
patch -s -f -N -p1 -i ../backports/v4.5_vb2_cpu_access.patch
patch -s -f -N -p1 -i ../backports/v4.4_gpio_chip_parent.patch
patch -s -f -N -p1 -i ../backports/v4.4_user_pages_flag.patch
Patched drivers/media/dvb-core/dvbdev.c
Patched drivers/media/v4l2-core/v4l2-dev.c
Patched drivers/media/rc/rc-main.c
make[1]: Leaving directory '/compile/Embedded-MediaDrivers/build/SYNOV1000/media_build/linux'
Preparing to compile for kernel version 4.4.302
WARNING: This is the V4L/DVB backport tree, with experimental drivers
         backported to run on legacy kernels from the development tree at:
                http://git.linuxtv.org/media-tree.git.
         It is generally safe to use it for testing a new driver or
         feature, but its usage on production environments is risky.
         Don't use it in production. You've been warned.
LIRC: Requires at least kernel 4.10.0
IR_IMON: Requires at least kernel 4.11.0
IR_PWM_TX: Requires at least kernel 4.8.0
CEC_CH7322: Requires at least kernel 4.16.0
CEC_CROS_EC: Requires at least kernel 9.255.255
V4L2_H264: Requires at least kernel 9.255.255
VIDEO_STK1160: Requires at least kernel 4.12.0
VIDEO_SOLO6X10: Requires at least kernel 4.5.0
VIDEO_IPU3_CIO2: Requires at least kernel 9.255.255
RADIO_WL128X: Requires at least kernel 4.13.0
VIDEO_MEM2MEM_DEINTERLACE: Requires at least kernel 9.255.255
VIDEO_MUX: Requires at least kernel 4.13.0
VIDEO_ASPEED: Requires at least kernel 4.9.0
VIDEO_IMX_MIPI_CSIS: Requires at least kernel 4.16.0
VIDEO_IMX8_JPEG: Requires at least kernel 4.18.0
VIDEO_OMAP3: Requires at least kernel 9.255.255
VIDEO_XILINX: Requires at least kernel 4.10.0
VIDEO_XILINX_CSI2RXSS: Requires at least kernel 4.13.0
VIDEO_HI556: Requires at least kernel 9.255.255
VIDEO_HI846: Requires at least kernel 4.10.0
VIDEO_HI847: Requires at least kernel 4.10.0
VIDEO_IMX208: Requires at least kernel 9.255.255
VIDEO_IMX214: Requires at least kernel 4.10.0
VIDEO_IMX219: Requires at least kernel 4.10.0
VIDEO_IMX258: Requires at least kernel 9.255.255
VIDEO_IMX274: Requires at least kernel 9.255.255
VIDEO_IMX290: Requires at least kernel 4.10.0
VIDEO_IMX319: Requires at least kernel 9.255.255
VIDEO_IMX334: Requires at least kernel 4.10.0
VIDEO_IMX335: Requires at least kernel 4.10.0
VIDEO_IMX355: Requires at least kernel 4.10.0
VIDEO_IMX412: Requires at least kernel 4.10.0
VIDEO_MT9V111: Requires at least kernel 4.10.0
VIDEO_NOON010PC30: Requires at least kernel 4.19.0
VIDEO_OG01A1B: Requires at least kernel 4.10.0
VIDEO_OV02A10: Requires at least kernel 9.255.255
VIDEO_OV08D10: Requires at least kernel 4.10.0
VIDEO_OV13858: Requires at least kernel 4.5.0
VIDEO_OV13B10: Requires at least kernel 4.10.0
VIDEO_OV2680: Requires at least kernel 4.10.0
VIDEO_OV2740: Requires at least kernel 9.255.255
VIDEO_OV5648: Requires at least kernel 4.10.0
VIDEO_OV5670: Requires at least kernel 9.255.255
VIDEO_OV5675: Requires at least kernel 9.255.255
VIDEO_OV5693: Requires at least kernel 4.10.0
VIDEO_OV7251: Requires at least kernel 9.255.255
VIDEO_OV772X: Requires at least kernel 9.255.255
VIDEO_OV8856: Requires at least kernel 9.255.255
VIDEO_OV8865: Requires at least kernel 4.10.0
VIDEO_OV9282: Requires at least kernel 4.10.0
VIDEO_OV9650: Requires at least kernel 9.255.255
VIDEO_OV9734: Requires at least kernel 4.10.0
VIDEO_RDACM20: Requires at least kernel 4.10.0
VIDEO_RDACM21: Requires at least kernel 4.10.0
VIDEO_CCS: Requires at least kernel 4.16.0
VIDEO_M5MOLS: Requires at least kernel 4.19.0
VIDEO_AK7375: Requires at least kernel 4.10.0
VIDEO_DW9714: Requires at least kernel 4.10.0
VIDEO_DW9768: Requires at least kernel 4.10.0
VIDEO_DW9807_VCM: Requires at least kernel 4.10.0
VIDEO_TDA1997X: Requires at least kernel 4.15.0
VIDEO_ADV7183: Requires at least kernel 4.19.0
VIDEO_ADV748X: Requires at least kernel 4.8.0
VIDEO_ISL7998X: Requires at least kernel 9.255.255
VIDEO_MAX9286: Requires at least kernel 4.19.0
VIDEO_I2C: Requires at least kernel 4.17.0
VIDEO_ST_MIPID02: Requires at least kernel 4.10.0
DVB_M88DS3103: Requires at least kernel 4.7.0
DVB_AF9013: Requires at least kernel 4.7.0
DVB_CXD2820R: Requires at least kernel 4.6.0
DVB_RTL2830: Requires at least kernel 4.7.0
DVB_RTL2832: Requires at least kernel 4.7.0
DVB_MN88443X: Requires at least kernel 4.9.0
SND_BT87X: Requires at least kernel 9.255.255
INTEL_ATOMISP: Requires at least kernel 9.255.255
VIDEO_HANTRO: Requires at least kernel 9.255.255
VIDEO_MAX96712: Requires at least kernel 4.19.0
VIDEO_ROCKCHIP_VDEC: Requires at least kernel 9.255.255
VIDEO_ZORAN: Requires at least kernel 4.18.0
VIDEO_IPU3_IMGU: Requires at least kernel 9.255.255
Created default (all yes) .config file
./scripts/make_myconfig.pl
[ ! -f "./config-mycompat.h" ] && echo "/* empty config-mycompat.h */" > "./config-mycompat.h" || true
perl scripts/make_config_compat.pl /compile/Embedded-MediaDrivers/build/SYNOV1000/linux-4.4.x ./.myconfig ./config-compat.h
creating symbolic links...
Kernel build directory is /compile/Embedded-MediaDrivers/build/SYNOV1000/linux-4.4.x
make -C ../linux apply_patches
make[1]: Entering directory '/compile/Embedded-MediaDrivers/build/SYNOV1000/media_build/linux'
Patches for 4.4.302 already applied.
make[1]: Leaving directory '/compile/Embedded-MediaDrivers/build/SYNOV1000/media_build/linux'
make -C /compile/Embedded-MediaDrivers/build/SYNOV1000/linux-4.4.x M=/compile/Embedded-MediaDrivers/build/SYNOV1000/media_build/v4l  modules
make[1]: Entering directory '/compile/Embedded-MediaDrivers/build/SYNOV1000/linux-4.4.x'
  Building modules, stage 2.
  MODPOST 0 modules
make[1]: Leaving directory '/compile/Embedded-MediaDrivers/build/SYNOV1000/linux-4.4.x'
./scripts/rmmod.pl check
found 0 modules
make: Leaving directory '/compile/Embedded-MediaDrivers/build/SYNOV1000/media_build/v4l'
root@07acd4a885d4:/compile/Embedded-MediaDrivers#
```

# Step 6: Installing the kernel modules
Move the compiled kernel modules to the /export folder, and then to your NAS. 
Create a folder with the result from `uname -r` in the `/usr/local/lib/modules` folder of your NAS. In my case, it was `/usr/local/lib/modules/4.4.302+`.

To load them, I used my `insert_modules.sh` script, which is a simplified version of the `hauppauge.sh` script from th0ma7's repo, but you can manually load them using the `insmod` linux command. I set up a scheduled task to automatically load them at boot.

These are the modules I loaded in the kernel for the dualHD (order is relevant):

1. mc.ko
2. rc-core.ko
3. videobuf-core.ko
4. videodev.ko
5. videobuf2-common.ko
6. videobuf2-v4l2.ko
7. videobuf2-memops.ko
8. videobuf2-vmalloc.ko
9.  dvb-core.ko
10. dvb-usb.ko
11. videobuf2-dvb.ko
12. dvb-pll.ko
13. tveeprom.ko
14. si2168.ko
15. em28xx.ko
16. em28xx-dvb.ko
17. si2157.ko

You'll see that something is wrong (missing modules, wrong module insert order) from the `dmesg` output, saying that it can't insert the module in the kernel due to unknown symbols, like:

```bash
[160210.957968] dvb_usb: Unknown symbol dvb_dmx_swfilter_raw (err 0)
[160210.964838] dvb_usb: Unknown symbol dvb_frontend_detach (err 0)
[160210.971773] dvb_usb: Unknown symbol dvb_net_release (err 0)
[160210.978227] dvb_usb: Unknown symbol dvb_unregister_frontend (err 0)
[160210.985570] dvb_usb: Unknown symbol dvb_register_frontend (err 0)
[160210.992588] dvb_usb: Unknown symbol dvb_create_media_graph (err 0)
[160210.999789] dvb_usb: Unknown symbol dvb_unregister_adapter (err 0)
```

# Step 7: Device firmware loading
In my case, I also needed to load the firmware for my device. In fact, `dmesg` was reporting the following error:
```bash
[164482.334837] si2168 8-0067: Direct firmware load for dvb-demod-si2168-d60-01.fw failed with error -2
[164482.345221] si2168 8-0067: Falling back to user helper
[164482.354917] si2168 6-0064: firmware file 'dvb-demod-si2168-d60-01.fw' not found
[164482.355050] si2168 8-0067: firmware file 'dvb-demod-si2168-d60-01.fw' not found
[164482.377077] si2157 9-0060: found a 'Silicon Labs Si2157-A30 ROM 0x50'
[164482.377134] si2157 10-0063: found a 'Silicon Labs Si2157-A30 ROM 0x50'
[164482.377154] si2157 10-0063: Direct firmware load for dvb_driver_si2157_rom50.fw failed with error -2
[164482.377155] si2157 10-0063: Falling back to user helper
[164482.380751] si2157 10-0063: error -11 when loading firmware
[164482.414631] si2157 9-0060: Direct firmware load for dvb_driver_si2157_rom50.fw failed with error -2
[164482.424993] si2157 9-0060: Falling back to user helper
```

This was due to missing firmware: I downloaded the correct firmwares from the [**CoreELEC repo**](https://github.com/CoreELEC/dvb-firmware/tree/master/firmware) 
and put it in the `/lib/firmware` folder of my NAS.
For my device, I needed the following firmware files:
- dvb-demod-si2168-d60-01.fw
- dvb-tuner-si2157-a30-01.fw (I had to rename it to dvb_driver_si2157_rom50.fw to make it work)


# Step 8: Enjoy!
To me, this "unsupported" device is now working flawlessly. I'm using it with Plex and it's working great. I also used it with TVHeadend and it worked great too.

In this repo's releases, I'm leaving the compiled kernel modules for my architecture and CPU family, in case someone needs them. I'm also leaving the modified files I used to compile them. Feel free to submit a pull request to add other configs and boards, if you successfully compiled the kmods!

From time to time, Synology may update the kernel, so you may have to wait for the sources to become available to update your system, should the kernel version may change. The good thing is that 4.4.302+ is the last 4.4 kernel, and it's EOL, so it should not change anymore. Synology also doesn't update major kernel versions, so you should be safe for a while.

Don't forget to star this repo if you found it useful!

