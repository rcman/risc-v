# risc-v

<BR>
Of course! Building your own Linux for real RISC-V hardware is a fantastic and highly educational project. It's the ultimate way to understand how an embedded system comes to life.
This guide will walk you through the entire process, from setting up a build environment to booting a custom kernel and root filesystem on your board.
A Word of Warning: This is an advanced topic. The process can be complex and requires patience. Flashing an incorrect bootloader can potentially "brick" your device, making it unbootable without special recovery tools (like a JTAG programmer). Always follow your specific board's documentation very carefully, especially for flashing instructions.
High-Level Overview
Building "Linux" for your hardware involves three main, independent components:
The Bootloader (usually U-Boot): A small program that runs first. Its job is to initialize the basic hardware (DRAM, clocks) and then load the Linux kernel into memory and execute it.
The Linux Kernel: The core of the operating system. It manages hardware, processes, memory, and provides the core services.
The Root Filesystem (rootfs): A collection of directories and files (/bin, /etc, /lib, etc.) that contains the user-space applications, libraries, and configuration files. This is what you interact with via the shell.
We will build these components on a powerful host computer (likely an x86_64 PC running Linux) using a cross-compiler, and then transfer the resulting files to your RISC-V hardware.
Prerequisites
A Linux Host Machine: A PC or VM running a standard Linux distribution like Ubuntu, Debian, or Fedora. This is where you will do all the compilation.
Your RISC-V Hardware: A board like the Sipeed Lichee RV, StarFive VisionFive 2, Canandian BeagleV, etc.
Board-Specific Documentation: This is CRITICAL. You need to know:
The exact name of your board's configuration (<board_name>_defconfig).
How to connect to its serial console (UART). This is your primary means of debugging and interaction. You'll likely need a USB-to-TTL serial adapter.
The boot media (SD card, eMMC, SPI flash).
The procedure for flashing the bootloader and other images.
A Cross-Compilation Toolchain: This is a set of tools (GCC, binutils, etc.) that run on your x86 host but generate code for the RISC-V architecture.
Setting up the Cross-Compiler
You have two main options:
Easy Way (Recommended): Install a pre-built toolchain from your distribution's package manager.
code
Bash
# For Debian/Ubuntu
sudo apt update
sudo apt install crossbuild-essential-riscv64

# For Fedora
sudo dnf install riscv64-linux-gnu-gcc
Hard Way (More Control): Build it yourself using tools like Crosstool-NG. This is usually unnecessary unless you have very specific requirements.
Once installed, you'll have commands like riscv64-linux-gnu-gcc. We will tell our build systems to use this toolchain via the CROSS_COMPILE environment variable.
Part 1: Building the Bootloader (U-Boot)
U-Boot is the de facto standard bootloader for most embedded boards.
Get the Source Code:
code
Bash
git clone https://github.com/u-boot/u-boot.git
cd u-boot
Configure for Your Board:
Find the defconfig file for your board in the configs/ directory. For example, for the StarFive VisionFive 2, it might be starfive_visionfive2_defconfig.
code
Bash
# Set environment variables for the build
export ARCH=riscv
export CROSS_COMPILE=riscv64-linux-gnu-

# Clean previous builds (good practice)
make mrproper

# Configure U-Boot for your specific board
# Replace <your_board>_defconfig with the correct file name!
make <your_board>_defconfig
Build U-Boot:
code
Bash
# The -j flag uses multiple cores to speed up compilation
make -j$(nproc)
Find the Output Files:
The build process creates several files. The most important ones are usually:
u-boot.bin or u-boot.itb: The main U-Boot image.
spl/u-boot-spl.bin: The Secondary Program Loader (SPL). On many systems, a tiny SPL is loaded first from a fixed location, and its job is to initialize DRAM and then load the full U-Boot.
Consult your board's documentation to know which files you need and where to flash them (e.g., to specific offsets on an SD card).
Part 2: Building the Linux Kernel
Now let's build the heart of the system.
Get the Source Code:
code
Bash
git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
cd linux
Configure the Kernel:
Just like with U-Boot, you need to start with a default configuration.
code
Bash
# Use the same environment variables
export ARCH=riscv
export CROSS_COMPILE=riscv64-linux-gnu-

# Start with a generic RISC-V default configuration
make defconfig

# (Optional but Recommended) Customize the configuration
make menuconfig
menuconfig opens a text-based UI where you can enable/disable drivers, filesystems (make sure EXT4 is enabled if your rootfs will be on an ext4 partition), networking features, etc. This is how you truly make it "your" Linux.
Build the Kernel:
code
Bash
# This will take a while
make -j$(nproc)
Find the Output Files:
The key outputs are:
arch/riscv/boot/Image: This is the compressed, bootable kernel image. This is the file you will give to U-Boot.
arch/riscv/boot/dts/<vendor>/<board>.dtb: The Device Tree Blob. This is a critical file that describes the hardware layout of your specific board to the kernel (e.g., where the UART is, how much memory there is). The kernel is generic; the DTB makes it specific to your hardware. You must use the correct DTB for your board.
Part 3: Building the Root Filesystem (Rootfs)
This is where you have the most choice, from easy to very complex.
Option A: The Easy Way (Use a Pre-built Rootfs)
For getting started, this is the fastest path. Many distributions provide minimal root filesystem tarballs for RISC-V.
Debian: https://wiki.debian.org/RISC-V#Rootfs
Fedora: https://fedoraproject.org/wiki/Architectures/RISC-V/Installing
You would download a .tar.gz file and extract it onto a partition on your SD card.
code
Bash
# Example for an SD card partition mounted at /mnt/riscv_root
# (Make sure the partition is formatted as ext4)
sudo tar -xpf /path/to/fedora-riscv-rootfs.tar.gz -C /mnt/riscv_root
Option B: The "Build Your Own" Way (Buildroot)
This is the recommended approach for a custom embedded system. Buildroot is a fantastic tool that automates the entire process of building a complete, minimal Linux system (toolchain, kernel, bootloader, and rootfs).
Get Buildroot:
code
Bash
git clone https://github.com/buildroot/buildroot.git
cd buildroot
Configure: Buildroot also has defconfig files for many boards.
code
Bash
# List available RISC-V configurations
make list-defconfigs | grep riscv

# Configure for your board (e.g., for the BeagleV Starlight)
make beagle_v_starlight_riscv64_defconfig
Customize (Optional):
code
Bash
make menuconfig
This menu lets you choose your kernel version, U-Boot version, and exactly which user-space packages (like htop, python, openssh) you want in your final image.
Build:
code
Bash
make -j$(nproc)
This will take a long time, as it will download and compile everything from scratch.
Find the Output:
The final images will be in output/images/. You'll typically find:
rootfs.tar
Image (the kernel)
.dtb file (the device tree)
U-Boot binaries
sdcard.img: A complete, bootable SD card image you can flash directly!
Option C: The Hardcore Way (Linux From Scratch)
If you truly want to build every single package by hand, follow the "Linux From Scratch" guide, adapting it for cross-compilation. This is a monumental task but provides the deepest possible understanding. This is not recommended for your first attempt.
Part 4: Putting It All Together and Booting
Let's assume you built the components manually (Parts 1 & 2) and are using a pre-built rootfs (Part 3A).
Prepare the SD Card:
Use a tool like gparted to create two partitions on your SD card.
Partition 1 (FAT32): A small (e.g., 100MB) boot partition.
Partition 2 (ext4): The rest of the card for the root filesystem.
Flash the Bootloader:
FOLLOW YOUR BOARD'S DOCUMENTATION. This step is the most dangerous. It often involves using the dd command to write the SPL and U-Boot binaries to specific raw offsets before the first partition. For example:
code
Bash
# !!! THIS IS AN EXAMPLE - DO NOT RUN BLINDLY !!!
# sudo dd if=spl/u-boot-spl.bin of=/dev/sdX bs=1k seek=32
# sudo dd if=u-boot.itb of=/dev/sdX bs=1k seek=1024
Copy Files:
Mount the FAT32 partition. Copy your kernel Image and your .dtb file to it.
Mount the ext4 partition. Extract your root filesystem tarball into it.
Boot!
Insert the SD card into your RISC-V board.
Connect your USB-to-serial adapter and open a terminal program (minicom, screen, putty) with the correct settings (e.g., 115200 baud rate).
Power on the board. You should see U-Boot's output on the serial console. It will likely stop at a U-Boot prompt (=>).
Tell U-Boot How to Boot Linux:
At the U-Boot prompt, you need to load the files and set the kernel command line.
code
Uboot
# Load the Device Tree from the SD card (partition 1) into memory
fatload mmc 0:1 ${fdt_addr_r} <your_board>.dtb

# Load the Kernel Image from the SD card into memory
fatload mmc 0:1 ${kernel_addr_r} Image

# Set the boot arguments. This is CRITICAL.
# - 'console=ttyS0,115200' tells Linux where to send its messages.
# - 'root=/dev/mmcblk0p2' tells Linux where the root filesystem is.
#   (This path may vary! It could be mmcblk1p2, etc.)
# - 'rootwait' tells it to wait for the SD card to be ready.
setenv bootargs 'console=ttyS0,115200 root=/dev/mmcblk1p2 rootwait'

# Boot the kernel! 'booti' is for the 'Image' format.
booti ${kernel_addr_r} - ${fdt_addr_r}
