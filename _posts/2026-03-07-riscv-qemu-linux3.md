---
title: "RISC-V Instruction Extensions on QEMU — Part 3: Adding a Custom Instruction"
date: 2026-03-03
categories:
  - riscv
  - qemu
  - linux
---

In the previous [post](https://utkugulgec.github.io/riscv/qemu/linux/2026/01/31/riscv-qemu-linux2.html) we cross-compiled a program, baked it into the initramfs, and ran it on our emulated RISC-V machine. Now comes the interesting part — actually extending the instruction set.

The goal for this post is to add a custom `ntt_mulmod` instruction to QEMU's RISC-V emulation. This instruction computes modular multiplication, which is a core operation in the Number Theoretic Transform (NTT). Instead of computing `(a * b) % q` in software, we want a single instruction that does it directly.

#### On this page

- [How QEMU Translates Instructions](#how-qemu-translates-instructions)
- [Instruction Encoding](#instruction-encoding)
- [Building QEMU from Source](#building-qemu-from-source)
- [Modifying the Source](#modifying-the-source)
- [Rebuilding QEMU](#rebuilding-qemu)
- [Writing the Test Program](#writing-the-test-program)
- [Verifying the Encoding](#verifying-the-encoding)
- [Running on QEMU](#running-on-qemu)

## How QEMU Translates Instructions

Before touching any code, it helps to understand what QEMU actually does with an instruction.

QEMU doesn't interpret instructions one by one. Instead, it uses a technique called **Tiny Code Generator (TCG)** — a JIT compiler that translates guest instructions (RISC-V in our case) into host machine code at runtime.

The pipeline looks like this:

```
Guest binary (RISC-V)
       ↓
   Decoder (insn32.decode)
       ↓
   Translator (trans_*.inc.c)
       ↓
   TCG IR (intermediate representation)
       ↓
   Host machine code (x86, ARM, etc.)
```

For adding a custom instruction, we only need to touch two parts of this pipeline:

- **`insn32.decode`** — tells the decoder what bit pattern corresponds to our instruction
- **`insn_trans/trans_ntt.inc.c`** — tells the translator what to do when it sees that instruction

That's it. No need to touch the JIT backend or anything lower.

## Instruction Encoding

RISC-V reserves four opcode spaces specifically for custom extensions:

| Opcode name | Binary      |
|-------------|-------------|
| custom-0    | `0001011`   |
| custom-1    | `0101011`   |
| custom-2    | `1011011`   |
| custom-3    | `1111011`   |

We will use **custom-0** (`0x0b`) with an R-type format, which takes two source registers and writes to one destination register.

The full encoding for `ntt_mulmod rd, rs1, rs2`:

```
 31      25 | 24  20 | 19  15 | 14  12 | 11   7 | 6      0
  funct7   |  rs2   |  rs1   | funct3 |   rd   | opcode
  0000000  |  rs2   |  rs1   |  000   |   rd   | 0001011
```

This encoding doesn't conflict with any standard RISC-V instruction, which is exactly the point of the custom opcode spaces.

## Building QEMU from Source

We need to build QEMU from source because we are going to modify it. The distro-provided QEMU binary we have used in Part I won't work here. So we need to clone it from GitHub.

Clone QEMU:

```bash
git clone https://github.com/qemu/qemu.git
cd qemu
```

Create a build directory and configure for RISC-V full system emulation:

```bash
mkdir build && cd build

../configure \
  --target-list=riscv64-softmmu \
  --enable-debug \
  --enable-debug-tcg
```

Build:

```bash
make -j$(nproc)
```

This will take a few minutes. Once done, verify the binary:

```bash
./riscv64-softmmu/qemu-system-riscv64 --version
```

## Modifying the Source

Now we make three small changes to the QEMU source. All files are under `target/riscv/`.

### 1. Declare the instruction — `insn32.decode`

Open `target/riscv/insn32.decode` and add at the bottom:

```
# NTT modular multiply (custom-0)
ntt_mulmod  0000000 ..... ..... 000 ..... 0001011 @r
```

The `@r` at the end tells the decoder to use the standard R-type format, which automatically extracts `rd`, `rs1`, and `rs2` from the instruction word.

### 2. Write the translation — `insn_trans/trans_ntt.inc.c`

Create a new file `target/riscv/insn_trans/trans_ntt.inc.c`:

```c
static bool trans_ntt_mulmod(DisasContext *ctx, arg_ntt_mulmod *a)
{
    TCGv src1  = tcg_temp_new();
    TCGv src2  = tcg_temp_new();
    TCGv prime = tcg_temp_new();
    TCGv tmp   = tcg_temp_new();
    TCGv dest  = tcg_temp_new();

    tcg_gen_mov_tl(src1, cpu_gpr[a->rs1]);
    tcg_gen_mov_tl(src2, cpu_gpr[a->rs2]);
    tcg_gen_movi_tl(prime, 3329);

    tcg_gen_mul_tl(tmp, src1, src2);
    tcg_gen_remu_tl(dest, tmp, prime);

    tcg_gen_mov_tl(cpu_gpr[a->rd], dest);

    tcg_temp_free(src1);
    tcg_temp_free(src2);
    tcg_temp_free(prime);
    tcg_temp_free(tmp);
    tcg_temp_free(dest);

    return true;
}
```

A few things worth noting here:

- We use `cpu_gpr[]` directly to read and write registers — this is the standard way in older QEMU versions
- `tcg_gen_mul_tl` and `tcg_gen_remu_tl` are TCG primitives for multiply and unsigned remainder
- The prime `3329` is hardcoded — this is the NTT-friendly prime used in Kyber/ML-KEM
- Returning `true` tells the decoder the instruction was handled successfully

### 3. Include the new file — `translate.c`

Open `target/riscv/translate.c` and find the block where other `trans_*.inc.c` files are included. Add our new file right after `trans_privileged.inc.c`:

```c
#include "insn_trans/trans_privileged.inc.c"
#include "insn_trans/trans_ntt.inc.c"       /* NTT custom instructions */
```

That's all the source changes needed.

## Rebuilding QEMU

From the build directory, run make again. Since only the RISC-V translation files changed, this is a fast incremental build:

```bash
cd build
make -j$(nproc)
```

## Writing the Test Program

Now we write a C program that invokes our new instruction using inline assembly. The `.insn r` directive lets us encode any R-type instruction directly without needing a custom assembler.

Create `~/riscv-qemu/programs/ntt_test.c`:

```c
#include <stdio.h>
#include <stdint.h>

#define NTT_PRIME 3329

static inline uint64_t ntt_mulmod(uint64_t a, uint64_t b)
{
    uint64_t result;
    asm volatile (
        ".insn r 0x0b, 0, 0, %0, %1, %2"
        : "=r"(result)
        : "r"(a), "r"(b)
    );
    return result;
}

int main(void)
{
    struct { uint64_t a; uint64_t b; } tests[] = {
        {1000, 2000},   /* normal case */
        {3328, 3328},   /* max inputs, tests overflow handling */
        {1,    3329},   /* b == prime, result should be 0 */
        {0,    1234},   /* zero input */
        {100,  100 },   /* small values */
        {1729, 1234},   /* arbitrary */
    };
    int n = sizeof(tests) / sizeof(tests[0]);

    int pass = 0, fail = 0;
    for (int i = 0; i < n; i++) {
        uint64_t a = tests[i].a;
        uint64_t b = tests[i].b;
        uint64_t expected = (a * b) % NTT_PRIME;
        uint64_t got = ntt_mulmod(a, b);
        int ok = (got == expected);
        printf("ntt_mulmod(%4lu, %4lu) = %4lu  expected %4lu  [%s]\n",
               a, b, got, expected, ok ? "PASS" : "FAIL");
        ok ? pass++ : fail++;
    }
    printf("\n%d passed, %d failed\n", pass, fail);
    return fail;
}
```

The `.insn r 0x0b, 0, 0, %0, %1, %2` line breaks down as:
- `0x0b` — custom-0 opcode
- first `0` — funct3
- second `0` — funct7
- `%0, %1, %2` — rd, rs1, rs2

Cross-compile it for RISC-V:

```bash
riscv64-linux-gnu-gcc \
  -static -O2 \
  -march=rv64gc -mabi=lp64d \
  ntt_test.c -o ntt_test
```

## Verifying the Encoding

Before running on QEMU, we can verify that the compiler actually emitted our custom instruction correctly by disassembling the binary.

```bash
riscv64-linux-gnu-objdump -d ntt_test | grep -A 80 "<main>:" | head -80
```

Since the disassembler doesn't know about our custom instruction, it won't print a mnemonic — it will just show the raw hex word. Look for it near the `mul` and `remu` instructions, since the compiler inlined our `ntt_mulmod` function:

```
103f0:	02d607b3          	mul	a5,a2,a3
103f4:	0327f7b3          	remu	a5,a5,s2
103f8:	00d6070b          	0xd6070b
```

The `0x00d6070b` at address `103f8` is our instruction. We can decode the 32-bit word to verify the fields are correct:

```bash
python3 -c "
x = 0x00d6070b
print(f'funct7  = {(x>>25)&0x7f:07b}')
print(f'rs2     = {(x>>20)&0x1f:05b} = x{(x>>20)&0x1f}')
print(f'rs1     = {(x>>15)&0x1f:05b} = x{(x>>15)&0x1f}')
print(f'funct3  = {(x>>12)&0x7:03b}')
print(f'rd      = {(x>>7)&0x1f:05b} = x{(x>>7)&0x1f}')
print(f'opcode  = {x&0x7f:07b}')
"
```

Output:

```
funct7  = 0000000
rs2     = 01101 = x13
rs1     = 01100 = x12
funct3  = 000
rd      = 01110 = x14
opcode  = 0001011
```

Everything matches our encoding spec — `funct7=0000000`, `funct3=000`, `opcode=0001011` (custom-0). The registers `x12 (a2)`, `x13 (a3)`, and `x14 (a4)` are just whatever the compiler allocated for the inlined function, which is expected.

## Running on QEMU

Repack the initramfs with the new binary (I created a directory for test programs under /riscv-qemu):

```bash
cd ~/riscv-qemu/rootfs
mkdir -p tmp && cd tmp
gzip -dc ../rootfs.img | cpio -idmv
cp ~/riscv-qemu/programs/ntt_test bin/
find . | cpio -o -H newc | gzip > ../rootfs.new.img
```

Boot with our custom QEMU:

```bash
~/qemu/build/riscv64-softmmu/qemu-system-riscv64 \
  -machine virt \
  -nographic \
  -bios default \
  -kernel ~/riscv-qemu/linux/arch/riscv/boot/Image \
  -initrd ~/riscv-qemu/rootfs/rootfs.new.img \
  -append "console=ttyS0"
```

Once the shell is ready, run:

```
/bin/ntt_test
```

The output should look like this:

```
ntt_mulmod(1000, 2000) = 2600  expected 2600  [PASS]
ntt_mulmod(3328, 3328) =    1  expected    1  [PASS]
ntt_mulmod(   1, 3329) =    0  expected    0  [PASS]
ntt_mulmod(   0, 1234) =    0  expected    0  [PASS]
ntt_mulmod( 100,  100) =   13  expected   13  [PASS]
ntt_mulmod(1729, 1234) = 3026  expected 3026  [PASS]

6 passed, 0 failed
```

All six test cases pass, including the edge cases — overflow at max inputs, zero operand, and input equal to the prime itself.

At this point we have a working custom instruction running inside a booted Linux kernel on our modified QEMU. The next step would be to implement the same instruction in RTL and bring it to actual hardware — which I am planning to work on in the future.

**Updated:** March 3, 2026
