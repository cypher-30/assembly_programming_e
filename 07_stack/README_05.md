# 05.asm: pushing and popping full 64-bit values, byte layout (little-endian)

## What the program does

```nasm
_start:
    mov rax, 0x1122334455667788   ; a full 64-bit value
    mov rbx, 0x1234               ; a small value (upper bytes are zero)

    push rax
    push rbx
    nop

    pop rbx          ; note: pops into the SAME registers it came from
    pop rax

    mov rax, 60
    xor rdi, rdi
    syscall
```

Like `01.asm` but (a) the values are chosen to show exactly **how many bytes** each push writes and **in what byte order**, and (b) the pops go back into the same registers, so it shows the "save and restore" pattern.

## Build and run

```
nasm -f elf64 -g 05.asm -o 05.o && ld 05.o -o 05
./05 ; echo $?        # -> 0, no output
```

## Trace (gdb `stepi`; state **before** each instruction)

`c8d0` = `0x7fffffffc8d0`.

| Instruction about to run | RSP | RAX | RBX | Notes |
|---|---|---|---|---|
| `mov rax, 0x1122…` | `c8d0` | 0 | 0 | assembler emits `movabs` (10-byte instruction) |
| `mov rbx, 0x1234` | `c8d0` | `1122334455667788` | 0 | |
| `push rax` | `c8d0` | `1122334455667788` | `1234` | |
| `push rbx` | `c8c8` | same | same | `[c8c8]` = `0x1122334455667788` |
| `nop` | `c8c0` | same | same | `[c8c0]` = `0x0000000000001234` |
| `pop rbx` | `c8c0` | same | same | about to read `[c8c0]` |
| `pop rax` | `c8c8` | same | same | RBX got `0x1234` (RSP +8) |
| `mov rax, 60` | `c8d0` | same | same | RAX got `0x1122334455667788` (RSP +8) |

(RAX and RBX look unchanged across the pops because they are restored with the very same values they held before: the point of save/restore.)

### Stack at the `nop`

```
 higher
   c8d0 | 0000000000000001 | argc
   c8c8 | 1122334455667788 | pushed from RAX  (first push -> deeper)
   c8c0 | 0000000000001234 | pushed from RBX  (second push -> top)   <== RSP
 lower
```

Each push consumed a full **8-byte** slot, even for the tiny value 0x1234: `push rbx` wrote `00 00 00 00 00 00 12 34` (as a 64-bit number), with the upper six bytes zero. A 64-bit `push` can't write "just 2 bytes"; the stack slot size in 64-bit mode is 8 bytes.

## Little-endian: how those 8 bytes sit in memory byte by byte

x86 stores the **least significant byte at the lowest address**. So the slot at `c8c8` holds, from low address to high:

```
 address: c8c8 c8c9 c8ca c8cb c8cc c8cd c8ce c8cf
 byte:     88   77   66   55   44   33   22   11
```

and the slot at `c8c0`:

```
 address: c8c0 c8c1 c8c2 c8c3 c8c4 c8c5 c8c6 c8c7
 byte:     34   12   00   00   00   00   00   00
```

When gdb dumps with `x/gx` (as one 64-bit number) it reassembles them back to `0x1122334455667788`. Dumping with `x/8xb` instead would show `88 77 66 …`. Same data, different view.

## Flow from start to end

1. Start: RSP = `c8d0`, stack holds only the kernel's argc/argv area.
2. Load two values into registers. The stack is untouched.
3. `push rax`: RSP −= 8 → `c8c8`; the 8 bytes of RAX are stored there.
4. `push rbx`: RSP −= 8 → `c8c0`; the 8 bytes of RBX are stored there. Stack depth is now 16 bytes.
5. `nop`: a pause point to inspect.
6. `pop rbx`: loads `[c8c0]` into RBX (gets `0x1234`: the **last pushed value comes first**), RSP += 8 → `c8c8`.
7. `pop rax`: loads `[c8c8]` into RAX (gets `0x1122334455667788`), RSP += 8 → `c8d0`.
8. Exit with status 0.

## The ordering lesson

Unlike `01.asm` (where the pops went into *different* registers and so appeared swapped), here the pops are in **reverse order of the pushes** (`push rax, push rbx` … `pop rbx, pop rax`). That is the correct way to save and restore registers: each register gets its own value back. This is the idiom used around function calls to preserve registers:

```nasm
push rax
push rbx
; ... code that clobbers RAX and RBX ...
pop rbx
pop rax
```

If the pops were `pop rax` then `pop rbx`, RAX would receive `0x1234` and RBX would receive `0x1122334455667788`, the values swapped. Notice that the program only restores correctly *because* of the reversal.

## Observations from running it

* Exit code `0`, no output.
* RSP: `c8d0 → c8c8 → c8c0` (pushes), `c8c0 → c8c8 → c8d0` (pops). Net change: 0.
* RAX and RBX hold their original values after the pops (confirmed in gdb) → proof that the save/restore round-trip works.
* After the pops, a memory dump at `RSP-16` still showed `0x1234` and `0x1122334455667788`. They are only logically discarded.
* The assembler turned `mov rax, 0x1122334455667788` into a 10-byte `movabs`, while `mov rbx, 0x1234` became a 5-byte `mov ebx, 0x1234` (writing to EBX zero-extends into RBX). Not stack-related, but visible in the disassembly.

## Takeaways

1. Every 64-bit push/pop moves exactly 8 bytes, regardless of how "big" the value is.
2. Multi-byte values are stored little-endian: low byte at the lowest address.
3. To restore registers, pop in the **reverse order** of the pushes.
4. Balanced push/pop leaves RSP unchanged.
