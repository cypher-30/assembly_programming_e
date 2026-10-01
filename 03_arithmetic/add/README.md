# ADD – Addition and Flags

In this folder I ran the `add` programs in GDB and checked the flags after each addition.

## How I ran them

```bash
nasm -f elf32 -g add1.asm -o add1.o
ld -m elf_i386 add1.o -o add1
gdb ./add1
```

In GDB I used `starti`, then `stepi` to go one instruction at a time, and `info registers eflags` to see the flags.

## What the flags mean

| Flag | Set to 1 when |
|------|---------------|
| CF (Carry) | the answer is too big for the register (unsigned) |
| PF (Parity) | the lowest 8 bits of the answer have an even number of 1s |
| AF (Auxiliary) | there is a carry out of the lowest 4 bits |
| ZF (Zero) | the answer is 0 |
| SF (Sign) | the highest bit of the answer is 1 (negative) |
| OF (Overflow) | the answer is wrong when the numbers are treated as signed |

IF is always on in normal programs, so I ignored it.

---

## add1.asm – `120 + 10` (8-bit)

```
  0111 1000  (120)
+ 0000 1010  (10)
= 1000 0010  (130) = 0x82
```

GDB: `AL = 0x82`, flags = `[ PF AF SF IF OF ]`

| Flag | Value | Why |
|------|-------|-----|
| CF | 0 | 130 fits in 8 bits (max 255) |
| PF | 1 | `1000 0010` has two 1s, which is even |
| AF | 1 | low 4 bits: 8 + 10 = 18, which is more than 15, so it carries |
| ZF | 0 | the answer is not 0 |
| SF | 1 | the top bit is 1 |
| OF | 1 | 120 and 10 are positive but the answer looks negative (-126) because 130 is bigger than 127 |

---

## add2.asm – `32000 + 500` (16-bit)

```
  0111 1101 0000 0000  (32000)
+ 0000 0001 1111 0100  (500)
= 0111 1110 1111 0100  (32500) = 0x7EF4
```

GDB: `AX = 0x7EF4`, flags = `[ IF ]` (all flags 0)

| Flag | Value | Why |
|------|-------|-----|
| CF | 0 | 32500 fits in 16 bits (max 65535) |
| PF | 0 | the low byte `1111 0100` has five 1s, which is odd |
| AF | 0 | low 4 bits: 0 + 4 = 4, no carry |
| ZF | 0 | the answer is not 0 |
| SF | 0 | the top bit is 0 |
| OF | 0 | 32500 is still below 32767, so the signed answer is correct |

---

## add3.asm – `0xFFFF + 1`, then `adc ax, 0`

### add ax, 1

```
  1111 1111 1111 1111  (65535)
+ 0000 0000 0000 0001  (1)
= 0000 0000 0000 0000  with 1 carried out
```

GDB: `AX = 0x0000`, flags = `[ CF PF AF ZF IF ]`

| Flag | Value | Why |
|------|-------|-----|
| CF | 1 | 65536 does not fit in 16 bits, so the extra 1 goes to CF |
| PF | 1 | the low byte is all 0s (zero 1s counts as even) |
| AF | 1 | low 4 bits: 15 + 1 = 16, so it carries |
| ZF | 1 | AX is 0 |
| SF | 0 | the top bit is 0 |
| OF | 0 | signed: -1 + 1 = 0, which is correct |

### adc ax, 0

`adc` adds the carry too: `0 + 0 + CF(1) = 1`

GDB: `AX = 0x0001`, flags = `[ IF ]` (all flags 0)

| Flag | Value | Why |
|------|-------|-----|
| CF | 0 | 1 fits easily, no new carry |
| PF | 0 | `0000 0001` has one 1, which is odd |
| AF | 0 | no carry from the low 4 bits |
| ZF | 0 | the answer is 1 |
| SF | 0 | the top bit is 0 |
| OF | 0 | 0 + 1 = 1, correct |

---

## What I learned

- CF is for **unsigned** numbers and OF is for **signed** numbers. In add1, CF = 0 but OF = 1.
- PF only checks the lowest 8 bits, even in 16-bit additions.
- `adc` is useful for adding big numbers in parts because it adds in the carry.

`adc4.asm` is empty so I did not run it.
