# Logic Instructions – AND, OR, XOR, NOT, TEST and Flags

In this folder I ran the logic programs in GDB and checked the flags after each instruction.

## How I ran them

```bash
nasm -f elf32 -g and.asm -o and.o
ld -m elf_i386 and.o -o and
gdb ./and
```

In GDB I used `starti`, then `stepi` to go one instruction at a time, and `info registers eflags` to see the flags.

## What the flags mean

| Flag | Set to 1 when |
|------|---------------|
| CF (Carry) | AND/OR/XOR/TEST always clear it; NOT never touches it |
| PF (Parity) | the lowest 8 bits of the result have an even number of 1s |
| ZF (Zero) | the result is 0 |
| SF (Sign) | the highest bit of the result is 1 (negative) |
| OF (Overflow) | AND/OR/XOR/TEST always clear it; NOT never touches it |

IF is always on in normal programs, so I ignored it.

---

## and.asm – `0b10101010 AND 0b11001100`

```
  1010 1010  (0xAA)
& 1100 1100  (0xCC)
= 1000 1000  (0x88, 136)
```

GDB: `AL = 0x88`, flags = `[ PF SF IF ]`

| Flag | Value | Why |
|------|-------|-----|
| CF | 0 | AND always clears CF |
| PF | 1 | `1000 1000` has two 1s (even) |
| ZF | 0 | the result is not 0 |
| SF | 1 | the top bit is 1 |
| OF | 0 | AND always clears OF |

## or.asm – `0b10101010 OR 0b11001100`

```
  1010 1010  (0xAA)
| 1100 1100  (0xCC)
= 1110 1110  (0xEE, 238)
```

GDB: `AL = 0xEE`, flags = `[ PF SF IF ]`

| Flag | Value | Why |
|------|-------|-----|
| CF | 0 | OR always clears CF |
| PF | 1 | `1110 1110` has six 1s (even) |
| ZF | 0 | the result is not 0 |
| SF | 1 | the top bit is 1 |
| OF | 0 | OR always clears OF |

## xor.asm – `0b10101010 XOR 0b11001100`

```
  1010 1010  (0xAA)
^ 1100 1100  (0xCC)
= 0110 0110  (0x66, 102)
```

GDB: `AL = 0x66`, flags = `[ PF IF ]`

| Flag | Value | Why |
|------|-------|-----|
| CF | 0 | XOR always clears CF |
| PF | 1 | `0110 0110` has four 1s (even) |
| ZF | 0 | the result is not 0 |
| SF | 0 | the top bit is 0 |
| OF | 0 | XOR always clears OF |

## test.asm – `0b10101010 TEST 0b00001111`

`test` does the same thing as `and` for the flags, but does not store the result anywhere — AL stays `0xAA`.

```
  1010 1010  (0xAA)
& 0000 1111  (0x0F)
= 0000 1010  (0x0A) -- used only to set flags, AL unchanged
```

GDB: `AL = 0xAA` (unchanged), flags = `[ PF IF ]`

| Flag | Value | Why |
|------|-------|-----|
| CF | 0 | TEST always clears CF |
| PF | 1 | the hidden result `0x0A` has two 1s (even) |
| ZF | 0 | the hidden result is not 0, so some bits overlap |
| SF | 0 | the top bit of the hidden result is 0 |
| OF | 0 | TEST always clears OF |

## not.asm – `NOT 0b10101010`

```
NOT 1010 1010 (0xAA) = 0101 0101 (0x55, 85)
```

GDB: `AL = 0x55`

I checked `info registers eflags` right before and right after `not al` and it was **unchanged both times** (`[ IF ]`, plus the reserved bit 1 that GDB always shows as set). This matches the instruction reference: **NOT modifies no flags at all** — it is the one logic instruction here that leaves every flag alone.

---

## What I learned

- AND, OR, XOR, and TEST all **clear CF and OF unconditionally** — those two flags only mean something for arithmetic (add/sub/mul), not for bitwise logic.
- TEST is exactly AND, except the result is thrown away — only the flags survive. Same for CMP vs SUB.
- NOT is the odd one out: it touches **no flags at all**, not even ZF, so you can't branch on whether a NOT produced zero without a separate TEST or CMP afterward.
- PF only ever looks at the low 8 bits, same as in the arithmetic programs.
