---
layout: single
title: "RISC-V Instruction Extensions on QEMU — Part 2: Running Programs"
categories: [riscv, qemu, linux]
tags: [emulation, kernel]
toc: true
---

In the previous [post](https://utkugulgec.github.io/riscv/qemu/linux/2026/01/12/riscv-qemu-linux.html) we booted Linux on a RISC-V machine emulation in QEMU. This time, we will run a simple C/C++ program on the target. There are several ways to do this. We will follow an easier path — compile a RISC-V program on the host, bake it into the initramfs, boot Linux on QEMU, and execute the program from userspace.

## Writing the Program

Create a `programs` folder inside your project directory:

```bash
mkdir -p ~/riscv-qemu/programs
```

Create a simple C program at `~/riscv-qemu/programs/hello.c`:

```c
#include <stdio.h>

int main(void)
{
    printf("Hello from RISC-V userspace!\n");
    return 0;
}
```

## Cross-Compiling for RISC-V

Now we cross-compile it for RISC-V. The `-static` flag is important here — it bundles all library dependencies into the binary so it runs self-contained inside the minimal initramfs environment without needing shared libraries.

```bash
riscv64-linux-gnu-gcc \
  -static \
  -O2 \
  -march=rv64gc \
  -mabi=lp64d \
  hello.c \
  -o hello
```

Verify the binary:

```bash
file hello
```

It should return something like:

```
hello: ELF 64-bit LSB executable, UCB RISC-V, version 1 (SYSV), statically linked
```

## Integrating into the Root Filesystem

Copy the binary into `usr/bin/` inside the rootfs — this directory is already in PATH so you can run the program by name without specifying the full path:

```bash
cp ~/riscv-qemu/programs/hello ~/riscv-qemu/rootfs/usr/bin/
chmod +x ~/riscv-qemu/rootfs/usr/bin/hello
```

Now rebuild the initramfs to include the new binary. Inside the rootfs directory run:

```bash
cd ~/riscv-qemu/rootfs
find . | cpio -o -H newc | gzip > ../rootfs.cpio.gz
```

This replaces the old initramfs.

## Running on QEMU

Start QEMU:

```bash
qemu-system-riscv64 \
  -machine virt \
  -nographic \
  -bios default \
  -kernel linux/arch/riscv/boot/Image \
  -initrd rootfs.cpio.gz \
  -append "console=ttyS0"
```

Once the shell is ready, run:

```
hello
```

You should see the output on the command line.

![Hello World!](/assets/images/riscv-qemu/riscvqemu3.png)
