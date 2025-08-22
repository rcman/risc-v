Installing Linux on the Banana Pi BPI-F3 from Scratch
This guide provides a comprehensive walkthrough for building and installing a custom Linux distribution on the Banana Pi BPI-F3, a single-board computer featuring the SpacemiT K1 RISC-V SoC. This process involves cross-compiling the bootloader (U-Boot), the Linux kernel, and preparing a root filesystem.

Who is this guide for?
This guide is intended for advanced users who are comfortable with the Linux command line, compiling software from source, and partitioning storage devices. If you are new to embedded Linux, you may want to start with a pre-built image first.

Part 1: Prerequisites
Before you begin, you will need the following hardware and software.

Hardware:
Banana Pi BPI-F3 Board: The target device.

microSD Card: A reliable card with at least 16 GB of storage.

microSD Card Reader: For writing to the card from your host computer.

5V/3A USB-C Power Supply: To power the board.

USB to TTL Serial Adapter: This is essential. It is the only way to see boot messages and debug issues during the initial setup. The BPI-F3 uses a 3.3V logic level.

Host Computer: A PC or laptop running a Linux distribution (like Ubuntu 20.04/22.04). A virtual machine can work, but a native installation is recommended for performance.

Software (on Host Computer):
You'll need to install a cross-compilation toolchain and several build tools.

# Update your package list
sudo apt update

# Install essential build tools
sudo apt install -y build-essential bc bison flex git libssl-dev libncurses5-dev \
gawk wget cpio unzip python3-distutils gcc-riscv64-linux-gnu \
binutils-riscv64-linux-gnu gdisk

Part 2: Building the Bootloader (U-Boot)
The boot process for the BPI-F3 involves multiple stages. We will compile U-Boot, which is the primary bootloader responsible for initializing the hardware and loading the Linux kernel.

Clone the U-Boot Source Code:
It's crucial to use the U-Boot source tree that has specific support for the BPI-F3.

git clone https://github.com/BPI-SINOVOIP/BPI-F3-Uboot.git -b BPI-F3-uboot-v2
cd BPI-F3-Uboot

Configure and Compile U-Boot:
The CROSS_COMPILE environment variable tells the build system to use the RISC-V toolchain.

# Set the cross-compiler prefix
export CROSS_COMPILE=riscv64-linux-gnu-

# Configure U-Boot for the BPI-F3
make bananapi-bpi-f3_defconfig

# Build U-Boot
make -j$(nproc)

After a successful build, you will have the necessary bootloader image: u-boot.itb.

Part 3: Building the Linux Kernel
Next, we will compile the Linux kernel.

Clone the Kernel Source Code:
Again, use the official repository for the BPI-F3 to ensure hardware support.

git clone https://github.com/BPI-SINOVOIP/BPI-F3-Kernel.git -b BPI-F3-kernel-6.2
cd BPI-F3-Kernel

Configure and Compile the Kernel:
This process compiles the kernel itself, its modules, and the Device Tree Blob (DTB) which describes the hardware to the kernel.

# Set environment variables for cross-compilation
export ARCH=riscv
export CROSS_COMPILE=riscv64-linux-gnu-

# Use the default configuration for the BPI-F3
make bananapi_bpi_f3_defconfig

# Build the kernel, modules, and device tree
make -j$(nproc)

The key output files will be arch/riscv/boot/Image (the kernel) and arch/riscv/boot/dts/spacemit/k1-bpi-f3.dtb (the device tree).

Part 4: Preparing the microSD Card
This is a critical step where we partition the SD card and copy the built files. This will erase all data on the card.

Identify the SD Card:
Insert the microSD card into your host computer. Use lsblk or dmesg to identify its device name (e.g., /dev/sdX). Be absolutely sure you have the correct device name to avoid wiping your hard drive.

Partition the Card:
We will create two partitions: a small FAT32 partition for the bootloader and kernel, and a larger ext4 partition for the root filesystem. We use gdisk for this.

# Replace /dev/sdX with your card's device name
sudo gdisk /dev/sdX

Inside gdisk, follow these steps:

Press o to create a new empty GUID Partition Table (GPT).

Press n to create the first partition.

Partition number: 1

First sector: (Press Enter for default)

Last sector: +256M (A 256MB boot partition)

Hex code: ea00

Press n to create the second partition.

Partition number: 2

First sector: (Press Enter for default)

Last sector: (Press Enter for default, to use the rest of the card)

Hex code: 8300 (Linux filesystem)

Press w to write the changes and exit.

Format the Partitions:

# Replace /dev/sdX with your card's device name
sudo mkfs.vfat -F 32 /dev/sdX1
sudo mkfs.ext4 /dev/sdX2

Install U-Boot:
The bootloader needs to be written to a specific offset before the first partition.

# Go to your U-Boot build directory
cd ../BPI-F3-Uboot

# Write the bootloader image to the SD card
# Replace /dev/sdX with your card's device name
sudo dd if=u-boot.itb of=/dev/sdX bs=1k seek=1025

Part 5: Preparing the Root Filesystem
A root filesystem (rootfs) contains all the system libraries, utilities, and applications (like bash, coreutils, apt, etc.). We will download a minimal Debian rootfs for the RISC-V architecture.

Download and Extract the Rootfs:

# Create a working directory
mkdir -p ~/bpi-f3-rootfs && cd ~/bpi-f3-rootfs

# Download a Debian base rootfs for riscv64
wget https://rcn-ee.com/rootfs/eewiki/minfs/debian-12-minimal-riscv64-2024-07-28.tar.xz

# Mount the ext4 partition
sudo mkdir -p /mnt/rootfs
sudo mount /dev/sdX2 /mnt/rootfs # Use your card's second partition

# Extract the rootfs to the partition
sudo tar -xvf debian-12-minimal-riscv64-2024-07-28.tar.xz -C /mnt/rootfs

Copy Kernel and Modules:
Now we copy the kernel we built into the rootfs and boot partition.

# Mount the boot partition
sudo mkdir -p /mnt/boot
sudo mount /dev/sdX1 /mnt/boot # Use your card's first partition

# Go to your kernel build directory
cd ../BPI-F3-Kernel

# Copy the kernel image and device tree to the boot partition
sudo cp arch/riscv/boot/Image /mnt/boot/
sudo cp arch/riscv/boot/dts/spacemit/k1-bpi-f3.dtb /mnt/boot/

# Install kernel modules into the root filesystem
sudo make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- modules_install INSTALL_MOD_PATH=/mnt/rootfs

Unmount the Partitions:
Always unmount cleanly before removing the card.

sync
sudo umount /mnt/boot
sudo umount /mnt/rootfs

Part 6: First Boot
Connect the Serial Adapter:
Connect your USB-to-TTL adapter to the BPI-F3's UART pins (TX, RX, GND). Open a serial terminal program (like minicom or screen) on your host computer, configured for 115200 baud, 8-N-1.

sudo minicom -b 115200 -o -D /dev/ttyUSB0

Boot the Board:
Insert the microSD card into the BPI-F3 and connect the USB-C power supply.

Watch the Serial Console:
You should see output from U-Boot, followed by the Linux kernel decompressing and booting. If everything worked, you will eventually be greeted with a login prompt.

Default Login: For many minimal images, the user is debian with password temppwd, or root with no password.

Congratulations! You have successfully built and installed a custom Linux system on your BPI-F3.
