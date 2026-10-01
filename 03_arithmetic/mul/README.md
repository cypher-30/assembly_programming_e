# MUL – Multiplication and Flags

In this folder I ran the `mul` programs in GDB and checked the flags after each multiplication.

## How I ran them

```bash
nasm -f elf32 -g mul1.asm -o mul1.o
ld -m elf_i386 mul1.o -o mul1
gdb ./mul1
```

In GDB I used `starti`, then `stepi`, and `info registers eflags` to see the flags. I used `p/x $eax` and `p/x $edx` to see the answer.

## How MUL works

`mul` uses unsigned numbers, and the answer is twice the size of the inputs:

| Size | Multiplies | Answer goes in |
|------|-----------|----------------|
| 8-bit | AL × number | AX |
| 16-bit | AX × number | DX:AX |
| 32-bit | EAX × number | EDX:EAX |

## Flags after MUL

- **CF and OF** are the only flags `mul` really sets, and they are always the same.
  - **0** means the top half (AH / DX / EDX) is 0, so the answer fits in the bottom half.
  - **1** means the top half is not 0, so you need both halves for the answer.
- **SF, ZF, AF, PF** are **undefined** after `mul`. GDB still shows a value for them, but it has no meaning, so I don't use them.

IF is always on in normal programs, so I ignored it.

---

## mul1.asm – `25 × 10` (8-bit)

```
25 × 10 = 250 = 0x00FA
AH = 0x00, AL = 0xFA
```

GDB: `AX = 0x00FA`, flags = `[ IF ]`

| Flag | Value | Why |
|------|-------|-----|
| CF | 0 | AH is 0. 250 fits in AL (max 255) |
| OF | 0 | same as CF |
| SF, ZF, AF, PF | 0 | undefined for `mul`, so these values mean nothing |

---

## mul2.asm – `3000 × 200` (16-bit)

```
3000 × 200 = 600000 = 0x000927C0
DX = 0x0009, AX = 0x27C0
```

GDB: `DX = 0x0009`, `AX = 0x27C0`, flags = `[ CF IF OF ]`

| Flag | Value | Why |
|------|-------|-----|
| CF | 1 | 600000 is bigger than 65535, so part of it went into DX (DX = 9) |
| OF | 1 | same as CF |
| SF, ZF, AF, PF | 0 | undefined for `mul` |

This is why the program saves both AX and DX. Using AX alone would give the wrong answer.

---

## mul3.asm – `100000 × 300000` (32-bit)

```
100000 × 300000 = 30000000000 = 0x00000006FC23AC00
EDX = 0x00000006, EAX = 0xFC23AC00
```

GDB: `EDX = 0x6`, `EAX = 0xfc23ac00`, flags = `[ CF IF OF ]`

| Flag | Value | Why |
|------|-------|-----|
| CF | 1 | the answer is bigger than 32 bits, so part of it went into EDX (EDX = 6) |
| OF | 1 | same as CF |
| SF, ZF, AF, PF | 0 | undefined. The top bit of EAX is 1 but SF is still 0, which shows SF does not follow the answer here |

---

## What I learned

- After `mul`, CF and OF tell you whether the answer needed the top half register.
- The other flags are undefined after `mul`, so I should not use them.
