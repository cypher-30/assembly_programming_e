# 06.asm: 32-bit push/pop, 4-byte slots and `ESP`

## What the program does

```nasm
BITS 32
_start:
    mov eax, 0x11223344
    mov ebx, 0x1234

    push eax
    push ebx
    nop

    pop ebx
    pop eax

    mov eax, 1       ; sys_exit
    xor ebx, ebx     ; status 0
    int 0x80
```

This is the 32-bit twin of `05.asm`: save two registers on the stack, then restore them in reverse order.

## Build and run (32-bit)

```
nasm -f elf32 -g 06.asm -o 06.o && ld -m elf_i386 06.o -o 06
./06 ; echo $?        # -> 0, no output
```

Note on the earlier build: the executable that was committed in the folder was built as **elf64** (`file` reported `ELF 64-bit`). It happens to run with exit code 0 anyway, because with only `push`/`pop` of 32-bit register names the 64-bit CPU just encodes them as 64-bit push/pop of the same registers, which stay balanced, and `int 0x80` still works via the 32-bit compat syscall. That is *not* a real 32-bit stack, though: slots would be 8 bytes, not 4. For this README I rebuilt it as `elf32` so ESP moves by 4. (By contrast `04.asm` built as elf64 crashes, as explained in README_04.)

## Trace (gdb `stepi`; state **before** each instruction)

`ba80` = `0xffffba80`.

| Instruction about to run | ESP | EAX | EBX | Notes |
|---|---|---|---|---|
| `mov eax, 0x11223344` | `ba80` | 0 | 0 | `[ba80]` = argc |
| `mov ebx, 0x1234` | `ba80` | `11223344` | 0 | |
| `push eax` | `ba80` | `11223344` | `1234` | |
| `push ebx` | `ba7c` | same | same | `[ba7c]` = `0x11223344` (ESP −4) |
| `nop` | `ba78` | same | same | `[ba78]` = `0x00001234` (ESP −4) |
| `pop ebx` | `ba78` | same | same | about to read `[ba78]` |
| `pop eax` | `ba7c` | same | same | EBX ← `0x1234`, ESP +4 |
| `mov eax, 1` | `ba80` | same | same | EAX ← `0x11223344`, ESP +4 |
| `xor ebx, ebx` | `ba80` | `1` | `1234` | EAX set to 1 for the exit call |

### Stack at the `nop`

```
 higher addresses
   ba80 | 00000001 | argc
   ba7c | 11223344 | pushed from EAX (first push)
   ba78 | 00001234 | pushed from EBX (second push)   <== ESP (top)
 lower addresses
```

Bytes in memory (little-endian), low → high address:

```
 ba78: 34 12 00 00        (0x00001234)
 ba7c: 44 33 22 11        (0x11223344)
```

## Flow from start to end

1. ESP = `ba80`. Kernel's argc is at the top.
2. Registers are loaded, with no stack activity.
3. `push eax`: ESP −= **4** → `ba7c`; store EAX.
4. `push ebx`: ESP −= 4 → `ba78`; store EBX. The stack has grown by 8 bytes in total.
5. `nop` (observation point).
6. `pop ebx`: read `[ba78]` → EBX = `0x1234` (the *last* thing pushed); ESP += 4 → `ba7c`.
7. `pop eax`: read `[ba7c]` → EAX = `0x11223344`; ESP += 4 → `ba80`.
8. Exit: EAX = 1 (syscall number), EBX = 0 (status) → `int 0x80`.

## Observations from running it

* Exit code `0`, no output.
* ESP walked `ba80 → ba7c → ba78` on the way down and `ba78 → ba7c → ba80` on the way up: **steps of 4**, back at the start.
* Both registers got their original values back (EAX=`0x11223344`, EBX=`0x1234` before they were reused for the exit syscall) because the pops are in reverse order of the pushes.
* Compared with `05.asm`: same logic, but every slot is half the size (4 vs 8 bytes) and ESP changes by 4 not 8.
* The final `xor ebx, ebx` is needed: in 32-bit Linux the exit status comes from EBX, and EBX would otherwise still hold `0x1234` (4660 & 0xFF = 52 would be the exit code).

## Takeaways

1. In 32-bit mode the stack unit is 4 bytes and the stack pointer is ESP.
2. Same rules as before: push = decrement then store, pop = load then increment, LIFO, balanced pushes/pops leave ESP unchanged.
3. Pop in reverse order to give each register its own value back.
4. Build 32-bit source as 32-bit (`-f elf32`, `ld -m elf_i386`) so the stack behaves the way the source says.
