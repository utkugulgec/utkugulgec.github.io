---
layout: single
title: "RISC-V Instruction Extensions on QEMU — Part 1: Booting Linux"
categories: [riscv, qemu, linux]
tags: [emulation, kernel]
toc: true
---

One of the nicest things about RISC-V is that it's meant to be extended. You're not locked into a fixed instruction set — you can add your own instructions if you have a workload that doesn't map well to generic CPU operations. That makes RISC-V a great playground for experimenting with ideas that sit somewhere between software and hardware.

In this project, I'm looking at instruction set extensions and whether they make sense for accelerating parts of the *Number Theoretic Transform (NTT)*, which is basically FFT generalized over finite integer fields. NTT shows up a lot in cryptography and polynomial math. For now, I am planning to keep things in QEMU (which is the first time I am using it) and maybe later move on to the actual hardware (FPGA).

The idea is not to build a full hardware accelerator, but to learn as much as possible by doing.

This post documents the setup work — getting Linux to boot on a modified RISC-V target — which is the foundation for experimenting with NTT-focused instruction extensions later on. We will emulate 64-bit RISC-V.

## Installing QEMU
First, install the distro-provided QEMU:
```bash
sudo apt install -y qemu-system-misc qemu-utils
```
Verify QEMU:

```bash
qemu-system-riscv64 --version
```

Install RISC-V Linux Cross Toolchain:
```bash
sudo apt install -y \
  gcc-riscv64-linux-gnu \
  g++-riscv64-linux-gnu \
  binutils-riscv64-linux-gnu
```

Verify the toolchain:
```bash
riscv64-linux-gnu-gcc --version
riscv64-linux-gnu-objdump --version
```

At this point it is a good idea to create a clean workspace. Mine is named **riscv-qemu**. Inside this workspace, we will download the Linux kernel.

## Building the RISC-V Linux Kernel
Clone the kernel (it can take some time):
```bash
git clone https://github.com/torvalds/linux.git
cd linux
```

After the download is completed, make sure to set following parameters for the build environment:
```bash
export ARCH=riscv
export CROSS_COMPILE=riscv64-linux-gnu-
```

Next we run make mrproper to return the kernel source tree to its unconfigured state and set the default configuration.
```bash
make mrproper
make defconfig
```

Build the kernel:
```bash
make -j$(nproc)
```

After this you should see the kernel Image in directory: `linux/arch/riscv/boot/`.
Make sure you can see "RISC-V 64-bit" in the returning string of the following command:
```bash
file arch/riscv/boot/Image
```
For me it is:
```
MS-DOS executable PE32+ executable (EFI application) RISC-V 64-bit (stripped to external PDB), for MS Windows
```

The "MS-DOS / PE32+" part looks surprising but it is normal — modern Linux kernels are built as UEFI-compatible PE images. The important part is "RISC-V 64-bit" which confirms the architecture is correct.

## Building BusyBox
Now, we need BusyBox. BusyBox will provide us with shell commands in the Linux environment we will boot on QEMU.

Before cloning, install the ncurses library which is required by the BusyBox configuration menu:
```bash
sudo apt install -y libncurses-dev
```

Clone BusyBox:
```bash
git clone https://github.com/mirror/busybox.git
```

Inside the busybox directory run the commands:
```bash
export ARCH=riscv
export CROSS_COMPILE=riscv64-linux-gnu-
make distclean
make defconfig
```

Before proceeding with the build, we need to enable static linking.
```bash
make menuconfig
```
Here, under "Settings" select "Build static binary (no shared libs)".
Alternatively, you can edit the **.config** file in the directory.
Just find the `# CONFIG_STATIC is not set` line and change it to:
`CONFIG_STATIC=y`.

Build BusyBox:
```bash
make -j$(nproc)
make install
```

After the build is complete, verify the BusyBox binary:
```bash
file busybox
```

If everything is done correctly, you should see something like:
```
busybox: ELF 64-bit LSB executable, UCB RISC-V, version 1 (SYSV), statically linked,
```

## Creating the Root Filesystem
Now we create the RootFS.
```bash
mkdir -p rootfs
cd rootfs
```

Copy BusyBox install:
```bash
cp -a ../busybox/_install/* .
```

Create required directories:
```bash
mkdir -p proc sys dev tmp
```

Create the init script:
```bash
nano init
```

Paste the following inside init:
```sh
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
mount -t devtmpfs none /dev
exec /bin/sh
```

Make it executable:
```bash
chmod +x init
```

initramfs provides a minimal userspace environment that the Linux kernel uses during early boot before the real root filesystem is available.

Create the initramfs archive:
```bash
find . | cpio -o -H newc | gzip > ../rootfs.cpio.gz
```

Up to this point we have:
- Installed QEMU
- Installed RISC-V toolchain
- Built the Linux kernel image
- Built BusyBox
- Created the root filesystem

Your basic file structure should look like this:

```
riscv-qemu
├── busybox
├── linux
└── rootfs
```

## Booting Linux with QEMU
Now, all we have to do is start QEMU with the right configuration. In your project directory, run the below command:
```bash
qemu-system-riscv64 \
  -machine virt \
  -nographic \
  -bios default \
  -kernel linux/arch/riscv/boot/Image \
  -initrd rootfs.cpio.gz \
  -append "console=ttyS0"
```

If during kernel boot QEMU hangs, you may have to change the bios configuration. This happened to me while verifying the steps in VirtualBox. On some systems QEMU bundles OpenSBI internally, while on others it must be provided explicitly via `-bios`. If you hit this issue, install OpenSBI:

```bash
sudo apt install -y opensbi
```

Verify the install:
```bash
dpkg -L opensbi | grep fw
```

You should see something like:
```
/usr/lib/riscv64-linux-gnu/opensbi/generic/fw_jump.elf
```

Then update the `-bios` flag in the QEMU command to point at `fw_jump.elf`:
```bash
qemu-system-riscv64 \
  -machine virt \
  -nographic \
  -bios /usr/lib/riscv64-linux-gnu/opensbi/generic/fw_jump.elf \
  -kernel linux/arch/riscv/boot/Image \
  -initrd rootfs.cpio.gz \
  -append "console=ttyS0"
```

If everything is OK, you should see Linux booting in the terminal.

![QEMU booting Linux](/assets/images/riscv-qemu/riscvqemu1.png)

![QEMU booting Linux](/assets/images/riscv-qemu/riscvqemu2.png)
