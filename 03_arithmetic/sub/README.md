# SUB – Subtraction and Flags

In this folder I ran the `sub` programs in GDB and checked the flags after each subtraction.

## How I ran them

```bash
nasm -f elf32 -g sub1.asm -o sub1.o
ld -m elf_i386 sub1.o -o sub1
gdb ./sub1
```

In GDB I used `starti`, then `stepi` to go one instruction at a time, and `info registers eflags` to see the flags.

## What the flags mean for subtraction

| Flag | Set to 1 when |
|------|---------------|
| CF (Carry) | we had to **borrow**, meaning the first number was smaller than the second (unsigned) |
| PF (Parity) | the lowest 8 bits of the answer have an even number of 1s |
| AF (Auxiliary) | the lowest 4 bits needed a borrow |
| ZF (Zero) | the answer is 0 |
| SF (Sign) | the highest bit of the answer is 1 (negative) |
| OF (Overflow) | the answer is wrong when the numbers are treated as signed |

IF is always on in normal programs, so I ignored it.

---

## sub1.asm – `50 - 80` (8-bit)

```
  0011 0010  (50)
- 0101 0000  (80)
= 1110 0010  = 0xE2 (-30)
```

GDB: `AL = 0xE2`, flags = `[ CF PF SF IF ]`

| Flag | Value | Why |
|------|-------|-----|
| CF | 1 | 50 is smaller than 80, so it had to borrow |
| PF | 1 | `1110 0010` has four 1s, which is even |
| AF | 0 | low 4 bits: 2 - 0, no borrow needed |
| ZF | 0 | the answer is not 0 |
| SF | 1 | the top bit is 1, so the answer is negative (-30) |
| OF | 0 | -30 fits in a signed byte (-128 to 127), so the signed answer is correct |

---

## sub2.asm – `1000 - 2000` (16-bit)

```
  0000 0011 1110 1000  (1000)
- 0000 0111 1101 0000  (2000)
= 1111 1100 0001 1000  = 0xFC18 (-1000)
```

GDB: `AX = 0xFC18`, flags = `[ CF PF SF IF ]`

| Flag | Value | Why |
|------|-------|-----|
| CF | 1 | 1000 is smaller than 2000, so it had to borrow |
| PF | 1 | the low byte `0001 1000` has two 1s, which is even |
| AF | 0 | low 4 bits: 8 - 0, no borrow needed |
| ZF | 0 | the answer is not 0 |
| SF | 1 | the top bit (bit 15) is 1, so the answer is negative |
| OF | 0 | -1000 fits in 16-bit signed, so the answer is correct |

These are the same flags as sub1, just with 16 bits.

---

## sub3.asm – `0 - 1`, then `sbb ax, 0`

### sub ax, 1

```
  0000 0000 0000 0000  (0)
- 0000 0000 0000 0001  (1)
= 1111 1111 1111 1111  = 0xFFFF (-1)
```

GDB: `AX = 0xFFFF`, flags = `[ CF PF AF SF IF ]`

| Flag | Value | Why |
|------|-------|-----|
| CF | 1 | 0 is smaller than 1, so it had to borrow |
| PF | 1 | the low byte `1111 1111` has eight 1s, which is even |
| AF | 1 | low 4 bits: 0 - 1 needs a borrow |
| ZF | 0 | the answer is not 0 |
| SF | 1 | the top bit is 1 (-1) |
| OF | 0 | 0 - 1 = -1 is correct |

### sbb ax, 0

`sbb` also subtracts the borrow: `0xFFFF - 0 - CF(1) = 0xFFFE` (-2)

GDB: `AX = 0xFFFE`, flags = `[ SF IF ]`

| Flag | Value | Why |
|------|-------|-----|
| CF | 0 | 0xFFFF is big enough to take 1 away, so no borrow |
| PF | 0 | `1111 1110` has seven 1s, which is odd |
| AF | 0 | low 4 bits: 15 - 1 = 14, no borrow |
| ZF | 0 | the answer is not 0 |
| SF | 1 | the top bit is still 1 (-2) |
| OF | 0 | -1 - 1 = -2 is correct |

---

## What I learned

- In subtraction, CF = 1 means a **borrow**: the first number was smaller.
- CF can be 1 while OF is 0. The unsigned answer is wrong, but the signed answer is correct.
- OF does not get set when both numbers have the same sign, because the answer cannot go out of range.
- `sbb` subtracts the borrow too, so it can be used to subtract big numbers in parts.
