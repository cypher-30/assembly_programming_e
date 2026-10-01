# DIV – Division and Flags

In this folder I ran the `div` programs in GDB and checked the flags after each division.

## How I ran them

```bash
nasm -f elf32 -g div1.asm -o div1.o
ld -m elf_i386 div1.o -o div1
gdb ./div1
```

In GDB I used `starti`, then `stepi`, and `info registers eflags` to see the flags. I used `p/x $eax` and `p/x $edx` to see the answer.

## How DIV works

`div` uses unsigned numbers. The number being divided is twice the size of the divisor:

| Divisor size | Divides | Quotient | Remainder |
|--------------|---------|----------|-----------|
| 8-bit | AX | AL | AH |
| 16-bit | DX:AX | AX | DX |
| 32-bit | EDX:EAX | EAX | EDX |

## Flags after DIV

**All the flags (CF, OF, SF, ZF, AF, PF) are undefined after `div`.** GDB shows values for them, but those values don't depend on the answer.

`div` does not use flags to report errors. If you divide by 0, or the quotient is too big for the register, the program crashes instead.

IF is always on in normal programs, so I ignored it.

---

## div1.asm – `100 ÷ 7` (8-bit)

```
100 ÷ 7 = 14 remainder 2
AL = 0x0E (14), AH = 0x02 (2)  →  AX = 0x020E
```

GDB: `AX = 0x020E`, flags = `[ AF IF ]`

| Flag | Value | Why |
|------|-------|-----|
| AF | 1 | undefined. It is 1 here, but this is not because of a carry |
| CF, OF, SF, ZF, PF | 0 | undefined, so the values mean nothing |

---

## div2.asm – `50000 ÷ 300` (16-bit)

```
DX:AX = 0:50000
50000 ÷ 300 = 166 remainder 200
AX = 0x00A6 (166), DX = 0x00C8 (200)
```

GDB: `AX = 0x00A6`, `DX = 0x00C8`, flags = `[ AF IF ]`

| Flag | Value | Why |
|------|-------|-----|
| AF | 1 | undefined, same as div1 |
| CF, OF, SF, ZF, PF | 0 | undefined |

DX must be set to 0 before dividing, because DX is the top half of the number being divided.

---

## div3.asm – `300000000 ÷ 1000` (32-bit)

```
300000000 ÷ 1000 = 300000 remainder 0
EAX = 0x000493E0 (300000), EDX = 0
```

GDB: `EAX = 0x493e0`, `EDX = 0x0`, flags = `[ AF IF ]`

| Flag | Value | Why |
|------|-------|-----|
| ZF | 0 | the remainder is 0, but ZF is still 0. This shows `div` does not set ZF from the answer |
| AF | 1 | undefined, same as before |
| CF, OF, SF, PF | 0 | undefined |

---

## What I learned

- All three divisions gave the **same flags** even though the answers were different, so the flags after `div` are not useful.
- To check the answer, I would need to use `cmp` or `test` after `div`.
- Dividing by 0, or getting a quotient that doesn't fit in the register, crashes the program instead of setting a flag.
