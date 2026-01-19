---
title: "Booting Linux on RISC-V using QEMU"
categories: [riscv, qemu, linux]
tags: [emulation, kernel]
toc: true
---

One of the nicest things about RISC-V is that it’s meant to be extended. You’re not locked into a fixed instruction set — you can add your own instructions if you have a workload that doesn’t map well to generic CPU operations. That makes RISC-V a great playground for experimenting with ideas that sit somewhere between software and hardware.

In this project, I’m looking at instruction set extensions and whether they make sense for accelerating parts of the Number Theoretic Transform (NTT), which is basically FTT generalized over integers. NTT shows up a lot in cryptography and polynomial math. For now, I am planning to keep things in QEMU (which is the first time I am using it) and maybe later on I will move on to the actual hardware, FPGA.

The idea is not to build a full hardware accelerator, but to learn as much as stuff by doing.

This post documents the setup work — getting Linux to boot on a modified RISC-V target — which is the foundation for experimenting with NTT-focused instruction extensions later on.

First, install the distro-provided QEMU
```bash
sudo apt install -y qemu-system-misc qemu-utils
```
Verify QEMU

```bash
qemu-system-riscv64 --version
```
