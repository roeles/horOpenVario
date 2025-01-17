*This file is part of horOpenVario*

*Copyright (C) 2017-2021  Kai Horstmann*

# Open Vario on Cubieboard 2 running Ubuntu or Debian
## horOpenVario

This repository is the main repository of [horOpenVario](https://github.com/hor63/horOpenVario.git).

This build system builds a complete Linux installation for a *Cubiebaord 2* with an Allwinner A20 (sun7i) SOC.   
It is intended to run [XCSoar](https://xcsoar.org)

This build system creates a SD card image containing a ready-to-run Linux system.
It includes
- The SD card image itself contains the /boot and root partition. 
- U-Boot binary image installed at the start of the image before the boot partition.
- A vanilla 5.10 LTS Linux kernel, currently the recent patch 58 compiled from sources
- Vanilla Ubuntu LTS 21.10 (Impish) / Debian Stable / Debian Testing installation with debootstrap, featuring:
  - Text mode only
  - Networking for USB tethering e.g. with Android, Ethernet, and multiple WiFi USB sticks (particularly Realtec).
  - Bluetooth components and driver for BT USB dongle are installed. However connecting to a Bluetooth hotspot is at this time completely manual.
  - Complete development package to compile XCSoar directly on the Cubieboard
  - **Working** OpenGL ES 2.0 Mali acceleration either with
    -  Closed-source MALI blob, and out-of-tree Mali driver
    -  Open-source LIMA driver in the kernel, and Mesa client side (Ubuntu Focal and Hirsute as well as Debian already bring a version of Mesa with working Lima driver)


## Checkout the repository

This repository requires a number of git submodules.
Therefore check it out either with
```
  git clone https://github.com/hor63/horOpenVario.git --recursive
```
or initialize and load the sub-modules separately.
```
  git clone https://github.com/hor63/horOpenVario.git
  cd horOpenVario
  git submodule init
  git submodule update
```

## Prerequisites and systems

My host system for buildng is Ubuntu 21.04 (Hirsute).
I also tested Debian Stable (Bullseye) and Testing (Bookworm) from a `debootstrap` installation with `chroot` under the Ubuntu kernel.  
I *highly* recomment to use the same distribution and version for host and target system.
This makes cross-compilation rather painless because compiler and runtime libs in the target system are fitting seamlessly when you cross-compile on the host.

This BSP is tested and works for Ubuntu and Debian OOTB.  
For Fedora, OpenSuse and the likes your mileage may vary.   
And don't even start with OpenEmbedded & Co. I have no idea how that may work :upside_down_face:

## Build

To perform a complete build run `./makenewimage.sh`.
Options are:
- `--no-pause`: Do not stop before every step of the build process but run as much as possible automatically. Stop only when a real input is required.
- `--with-mali`: Load the `mali` module for the closed-source Mali blob at boot time.  
  By default start the system with the open-source `lima` module.   
  Please note that even with --with-mali the OSS Lima kernel module is being built and installed. It is just blacklisted. Instead the cole-source Mali module is loaded by default.
  Also the proprietary Mali blob is provided in the root directory as .deb image when you later decide to switch from Lima to Mali.

Without `--no-pause` the script will stop at any step and print a text explaining what comes next. Continue just hitting `Enter`.  
Be careful not blindly hitting enter each time the script stops however. There are questions and actual input required along the way.  

Questions which will be asked or options to be chosen are:
- Select the Ubuntu version to be installed (Hirsute (21.04) version is default. Required for the most recent XCSoar due to GCC and Lua version requirements).
- If you re-use a previously downloaded base installation tarball when you run `./makenewimage.sh` multiple times, or download it again (default is re-use).
- `root` password of the new system. The system will have by default a working root user. Use it at your own risk.
- Use of an APT proxy. It requires a working `apt-proxy` installation in your network (`apt-proxy-ng` is proven to be *not* reliable). Default is not using a cache.  
  *Please note*: The proxy setting is being written to the APT configuration on the target image into ´/etc/apt/apt.conf.d/00aptproxy´. You may want to delete it once you have the Cubieboard up and running. In case you provide a real hostname, and valid port, and your Cubie can reach that machine you can continue using the cache.
- Host name of the target machine. Chose a unique name within your network. No default. You have to enter a real name.
- (text based) network management:
  - **Network manager**. Use `nmtui` to configure WiFi networks. Default.
  - No network management at all. You need to install additional packages yourself, and perform the configuration manually.
- If you want to install development tools for native building XCSoar (and other stuff) on the Cubieboard. Default Yes.  
  **Note**: Debian and Ubuntu have the nasty habit to link shared libraries with an absolute path to the actual file.
  Example is the symbolic link `/lib/arm-linux-gnueabihf/libz.so` -> `/lib/arm-linux-gnueabihf/libz.so.1.2.11`. When you are cross-compiling with a root file system of the
  target system like `/home/foo/horOpenVario/sdcard` the symbolic link of `lib/libz.so` *should* point to
  `/home/foo/horOpenVario/sdcard/lib/arm-linux-gnueabihf/libz.so.1.2.11` but with the absolute path it points to `/lib/arm-linux-gnueabihf/libz.so.1.2.11`. But there is noting.  
  Therefore symbolic links are created from `/lib/arm-linux-gnueabihf` to your SD card image. This has not adverse side effect which I am aware of. But be warned that the script does stuff to your `/lib/arm-linux-gnueabihf` directory.
- Interactively select the timezone in which the Cubie is located.
- Interactively select the locale(s) you want to install. Select *at least* `en_US.UTF-8`. Otherwise you may get funny error messages, particularly from Python based stuff.
- Select the keyboard type, and layout (English, German...).

Then copy the newly created `sd.img` file onto the SD card e.g. with 
```
dd if=sd.img of=/dev/mmcblk0 BS=1M
```
**Note**: The device name of the SD card can vary. Best is to run `tail -f /var/log/syslog` in one command window after you insert the SD card.
On some system SD cards can be listed as SCSI disk, e.g. `/dev/sdc`. Take great care to copy the SD card image with `dd` into the correct device.  
I once overwrote accidently the USB drive from which I run my build Linux system! :facepalm:

To run the freshly installed Ubuntu system on the target Cubieboard 2 simply insert the SD card into the Cubieboard, connect USB keyboard and a HDMI monitor.  
Power it up.

**Enjoy, have fun with it.**

# Work with the image and compile XCSoar

## Switch between Mali and Lima

You can switch between open-source Lima and closed source Mali blob on the target system. Simply run `/switch-to-mali.sh` or `/switch-to-lima.sh` as root, and re-boot.

## Compile XCSoar
When you install the development package on the image you can build mainline XCSoar OOTB on the Cubie. Just be patient but it works.

### For Mali
When the Mali driver is active you simply run `make` (optionally with DEBUG=n and -j2). `make` finds the /dev/mali device, and compiles against the Mali blob.

### For Lima
As a first test you may want [download KMSCUBE from freedesktop.org](https://gitlab.freedesktop.org/mesa/kmscube.git) and compile it to verify if hardware acceleration with Lima is actually active and working. The program spits out a lot of information about EGL and OPENGL ES 2.0.

When you have Lima active you can compile XCSoar with `make ENABLE_MESA_KMS=y`.
You need to start it for the DRI device /dev/card1 like on PI 4.

### For X11 with Lima acceleration (proof of concept)
If you feel frisky you can install a graphical desktop (I tested `apt-get install lubuntu-desktop`), and compile XCSoar with `make OPENGL=y GLES2=y` for X11 with Lima acceleration.

### Cross compilation

The generated SD card image on your disk can also serve as your root file system for cross compilation. 
After building the SD card image sd.img you can mount the image with `mountSDImage.sh`. It is mounted under sdcard/

Cross-compiling with `make TARGET=CUBIE CUBIE=<path to the root file system>` by default compiles for the Mali blob. In the recent HEAD version the build automatically deals with different native EGL window structures for the old headers for the 3.4 kernel, and the mainline kernel package from Bootlin.

You can now cross-compile the most recent version also for Mesa/KMS with `make TARGET=CUBIE CUBIE=<path to the root file system> ENABLE_MESA_KMS=y`.
