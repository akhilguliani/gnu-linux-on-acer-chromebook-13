GNU/Linux on Acer Chromebook 13 (aka CB5-311 or nyan-big)
=========================================================

Prerequisites
-------------

Build system: install dtc (device-tree-compiler), cgpt, uboot-tools, vboot-utils and Chrome OS developer keys (vboot-kernel-utils / chromeos-devkeys / etc) and a cross-compilation toolchain for ARM if building on another platform.


Developer mode on Chromebook
----------------------------

1) Power on while holding down the ESC and Refresh keys.

2) Enable developer mode by pressing Ctrl-D on the recovery screen. WARNING: this will erase user data from the Chromebook.

3) Enable booting from external media by running `sudo crossystem dev_boot_usb=1` from a terminal once the the system has booted.


Build a custom kernel image
---------------------------

On the build system:

    git clone https://chromium.googlesource.com/chromiumos/third_party/kernel -b chromeos-3.10
    cd kernel
   
Get the USB firmware (from [this overlay](https://chromium.googlesource.com/chromiumos/overlays/board-overlays/+/master/overlay-nyan/sys-kernel/xhci-firmware/xhci-firmware-2014.11.06.00.00.ebuild)):

    wget http://commondatastorage.googleapis.com/chromeos-localmirror/distfiles/xhci-firmware-2014.11.06.00.00.tbz2
    tar xf xhci-firmware-2014.11.06.00.00.tbz2
    cp -R lib/firmware/ .
    rm -R ./lib/firmware/
   
Prepare the config file:

    ./chromeos/scripts/prepareconfig chromeos-tegra
    WIFIVERSION=-3.8 ARCH=arm CROSS_COMPILE=arm-none-eabi- make menuconfig
    
The WIFIVERSION variable is required to build the drivers in `drivers/net/wireless-3.8`; the mwifiex driver in /drivers/net/wireless/ does not support the Chromebook's adapter. The ARM and CROSS\_COMPILE variables are only required if cross-compiling. CROSS\_COMPILE should be changed to the correct prefix for your toolchain.

Set `Device Drivers > Generic Driver Options > External firmware blobs to build into the kernel binary` to `nvidia/tegra124/xusb.bin`.

Change any other required settings. I've had some issues with HDCP negociation timing out when connecting a HDMI monitor, I would recommend disabling it.

Now the kernel can be built:

    WIFIVERSION=-3.8 ARCH=arm CROSS_COMPILE=arm-none-eabi- make zImage
    WIFIVERSION=-3.8 ARCH=arm CROSS_COMPILE=arm-none-eabi- make modules

If the build fails because of compiler warnings treated as errors (happens especially when you enable additional options in the kernel config), disable ``CONFIG_ERROR_ON_WARNING`` (*Kernel hacking -> Treat compiler warnings as errors* in `menuconfig`).

Build the DTB file:

    WIFIVERSION=-3.8 ARCH=arm CROSS_COMPILE=/usr/bin/arm-none-eabi- make tegra124-nyan-big-<board revision>.dtb

Available board revisions can be seen using the command

    ls arch/arm/boot/dts/tegra124-nyan-big-*.dts

To find out which one your device has, run the following command in the Chrome OS shell:

    cat /proc/device-tree/model

Build a Flattened Image Tree using the configuration file I provide:

    wget https://raw.githubusercontent.com/lgeek/gnu-linux-on-acer-chromebook-13/master/nyan-big-fit.cfg
    mkimage -f nyan-big-fit.cfg nyan-big-kernel
    
Create a bootable vboot image:

    echo "root=/dev/mmcblk1p2 rootwait rw noinitrd" > ./cmdline
    echo blah > dummy.txt
    vbutil_kernel --arch arm --pack kernel.bin --keyblock <PATH TO KEYBLOCK> --signprivate <PATH TO PRIVATE KEY> --version 1 --config cmdline --vmlinuz nyan-big-kernel --bootloader dummy.txt
    
    Jetson TK1
    echo "console=ttyS0,115200n8 console=tty1 no_console_suspend=1 lp0_vec=2064@0xf46ff000 video=tegrafb mem=1862M@2048M memtype=255 ddr_die=2048M@2048M section=256M pmuboard=0x0177:0x0000:0x02:0x43:0x00 vpr=151M@3945M tsec=32M@3913M otf_key=c75e5bb91eb3bd947560357b64422f85 usbcore.old_scheme_first=1 core_edp_mv=1150 core_edp_ma=4000 tegraid=40.1.1.0.0 debug_uartport=lsport,3 power_supply=Adapter audio_codec=rt5640 modem_id=0 android.kerneltype=normal usb_port_owner_info=0 fbcon=map:1 commchip_id=0 usb_port_owner_info=0 lane_owner_info=6 emc_max_dvfs=0 touch_id=0@0 tegra_fbmem=32899072@0xad012000 board_info=0x0177:0x0000:0x02:0x43:0x00 root=/dev/mmcblk1p2 rootwait rw" > ./cmdline
echo blah > dummy.txt
vbutil_kernel --arch arm --pack kernel.bin --keyblock /usr/share/vboot/devkeys/kernel.keyblock --signprivate /usr/share/vboot/devkeys/kernel_data_key.vbprivk --version 1 --config cmdline --vmlinuz jetson-kernel --bootloader dummy.txt


Partition an SD card
--------------------

    sudo cgpt create <MMC BLOCK DEVICE>
    sudo cgpt add -b 34 -s 32768 -P 1 -S 1 -t kernel <MMC BLOCK DEVICE> # 16 MB kernel image partition
    sudo cgpt add -b 32802 -s <ROOT PARTITION SIZE in 512B sectors> -t rootfs <MMC BLOCK DEVICE>
    
cgpt doesn't seem to create a protective MBR. If one is not already in place, it can be created with:

    sudo gdisk <MMC BLOCK DEVICE> # and enter command w

Copy data to the SD card
------------------------

    sudo dd if=./kernel.bin of=<MMC BLOCK DEVICE>p1
    sudo mkfs.ext4 <MMC BLOCK DEVICE>p2
    sudo mount <MMC BLOCK DEVICE>p2 /mnt/

Now you're ready to copy a GNU/Linux rootfs to the root partition. I use NVIDIA's [L4T](https://developer.nvidia.com/platform-software-development) rootfs because it's ready to run NVIDIA's Xorg driver. If you decide to use it as well, you'll need the 'Sample file system' and 'Driver Package' archives. Don't forget to install the kernel modules for the kernel you've just built:

    sudo INSTALL_MOD_PATH=/mnt/ WIFIVERSION=-3.8 ARCH=arm CROSS_COMPILE=/usr/bin/arm-none-eabi- make modules_install

Finally, unmount the rootfs, move the SD card to the Chromebook and boot the new system by pressing Ctrl-U at the boot warning screen.


Misc notes
----------

* The packed kernel and the rootfs can presumably also be copied to the internal eMMC, but I have not tried that yet.
* The touchpad needs the `xserver-xorg-input-synaptics` driver (which has multitouch support) to work.
* ~~WiFi seems to occasionally disconnect when idle for a long time if [NetworkManager](https://wiki.gnome.org/Projects/NetworkManager) is used, [wicd](http://wicd.sourceforge.net/) seems reliable.~~ NetworkManager is now working reliably for me. Older versions of the mwifiex driver (including the one shipped with my device) crash when NetworkManager manages the `uap0` device.
* OpenGL and XV hardware acceleration work correctly with the kernel interfaces exposed by the Chrome OS kernel; I have not yet tested CUDA.
* The Chromebook's keyboard has a slightly different layout compared to standard keyboards: the function keys (from *Back* to *Volume Up*) map to F1 to F10 and there are no F11 and F12 keys, so any standard shortcuts using these keys must be remapped. There are no Insert, Delete, Home, End, Page Up and Page Down keys, which takes a while to get used to.
* The Tegra K1 SoC in my Chromebook is a lower speed grade than the one used by [Jetson TK1](https://developer.nvidia.com/jetson-tk1), the maximum core frequency allowed by this kernel is 2.01 GHz.

