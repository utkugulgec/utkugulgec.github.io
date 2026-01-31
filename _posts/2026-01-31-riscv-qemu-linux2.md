---
layout: single
title: "RISC-V Instruction Extensions on QEMU — Part 2: Running C++ Programs"
categories: [riscv, qemu, linux]
tags: [emulation, kernel]
toc: true
---

In the previous [post](https://utkugulgec.github.io/riscv/qemu/linux/2026/01/12/riscv-qemu-linux.html) we booted Linux on a RISC-V machine emulation in QEMU. This time, we will run a simple C/C++ program on the target. There are several ways to do this. We will follow an easier path. Simply, compile a RISC-V C/C++ program on the host, bake it into the initramfs, boot Linux on QEMU, and execute the program from userspace.

Firstly, create a simple C/C++ code. I created a folder inside my project directory called programs.
```bash
~/riscv-project/programs/hello.c
```
```c
#include <stdio.h>

int main(void)
{
    printf("Hello from RISC-V userspace!\n");
    return 0;
}
```

Now we cross-compile it for RISC-V:
```bash
riscv64-linux-gnu-gcc \
  -static \
  -O2 \
  hello.c \
  -o hello
```
Verify the binary:
```bash
file hello
```

It should return something like:
```bash
ELF 64-bit LSB executable, UCB RISC-V, statically linked
```

Now we integrate the binary into the root file system. Copy the binary inside a place you create for user programs and ensure it is executable with chmod +x:
```bash
cp ~/riscv-project/programs/hello ~/riscv-project/rootfs/usr/bin/
```
Now we need to rebuild initramfs. Inside rootfs directory run below command:
```bash
find . | cpio -o -H newc | gzip > ../rootfs.cpio.gz
```
This replaces the old initramfs.

Finally we start QEMU.
```bash
qemu-system-riscv64 \
-machine virt \
-nographic \
-bios default \
-kernel linux/arch/riscv/boot/Image \
-initrd rootfs.cpio.gz \
-append "console=ttyS"
```
Once the shell is ready, run the binary from the directory you put it. You should see the output on command line.
![Hello World!](/assets/images/riscv-qemu/riscvqemu3.png)



