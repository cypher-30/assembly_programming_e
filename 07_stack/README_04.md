# 04.asm: the stack frame from 02.asm, in 32-bit (`ESP` / `EBP`)

## What the program does

```nasm
BITS 32
_start:
    call work
    mov eax, 1          ; sys_exit (32-bit numbering)
    xor ebx, ebx        ; status 0
    int 0x80            ; 32-bit system call

work:
    push ebp
    mov ebp, esp
    sub esp, 8                     ; room for two 4-byte locals

    mov dword [ebp - 4], 100
    mov dword [ebp - 8], 200

    mov eax, [ebp - 4]
    mov ebx, [ebp - 8]
    nop

    mov esp, ebp
    pop ebp
    ret
```

Same design as `02.asm`, but in **32-bit mode**. Everything that was 8 bytes becomes 4 bytes.

## Build and run (must be 32-bit!)

```
nasm -f elf32 -g 04.asm -o 04.o && ld -m elf_i386 04.o -o 04
./04 ; echo $?        # -> 0, no output
```

The `BITS 32` directive only tells NASM to *encode* 32-bit instructions; the object format must be `elf32` too. I confirmed that building it as `elf64` **segfaults (exit 139)**. The reason: in 64-bit mode `push ebp` and `call` still push 8-byte slots, and `mov esp, ebp` writes a 32-bit value into ESP, which zero-extends into RSP and throws away the upper half of the stack address (`0x7fff…`). RSP then points at a bogus low address, so `pop ebp` / `ret` read unmapped memory. `file 04` on my rebuilt binary confirms `ELF 32-bit … Intel 80386`.

## What changes from 64-bit to 32-bit

| | 64-bit (02.asm) | 32-bit (04.asm) |
|---|---|---|
| Stack pointer / frame pointer | RSP / RBP | **ESP / EBP** |
| Size of one stack slot | 8 bytes | **4 bytes** |
| `push`/`call` change SP by | −8 | **−4** |
| `pop`/`ret` change SP by | +8 | **+4** |
| Exit | `syscall` (rax=60) | `int 0x80` (eax=1) |
| Locals | `[rbp-8]`, `[rbp-16]` | `[ebp-4]`, `[ebp-8]` |

## Trace (gdb `stepi`; state **before** each instruction)

`ba80` means `0xffffba80`.

| Instruction about to run | ESP | EBP | Notes |
|---|---|---|---|
| `call work` | `ba80` | 0 | top of stack = argc |
| `push ebp` | `ba7c` | 0 | `call` pushed return addr `0x8049005` at `[ba7c]` (−4) |
| `mov ebp, esp` | `ba78` | 0 | old EBP (0) saved at `[ba78]` (−4) |
| `sub esp, 8` | `ba78` | `ba78` | EBP = frame anchor |
| `mov dword [ebp-4], 100` | `ba70` | `ba78` | ESP −8: space for 2 locals |
| `mov dword [ebp-8], 200` | `ba70` | `ba78` | `[ba74]` = 100 |
| `mov eax, [ebp-4]` | `ba70` | `ba78` | `[ba70]` = 200 |
| `mov ebx, [ebp-8]` | `ba70` | `ba78` | EAX = 100 |
| `nop` | `ba70` | `ba78` | EBX = 200 |
| `mov esp, ebp` | `ba70` | `ba78` | |
| `pop ebp` | `ba78` | `ba78` | locals released (ESP +8) |
| `ret` | `ba7c` | **0** | EBP restored |
| `mov eax, 1` (back in `_start`) | `ba80` | 0 | `ret` popped `0x8049005` into EIP |

### Stack picture at the `nop`

```
 higher addresses
   ba80 | 00000001   | argc
   ba7c | 08049005   | return address (pushed by call)
   ba78 | 00000000   | saved EBP (pushed by push ebp)     <== EBP
   ba74 | 00000064   | local #1 = 100  = [ebp-4]
   ba70 | 000000c8   | local #2 = 200  = [ebp-8]          <== ESP
 lower addresses
```

Each slot is 4 bytes, so the frame (saved EBP + 2 locals) is 12 bytes, plus the 4-byte return address above it.

## Flow from start to end

1. `_start` starts with ESP = `ba80`.
2. `call work`: pushes the 4-byte return address; ESP `ba80 → ba7c`; jumps to `work`.
3. `push ebp`: saves the caller's frame pointer; ESP → `ba78`.
4. `mov ebp, esp`: EBP = `ba78`. `[ebp]` = saved EBP, `[ebp+4]` = return address, `[ebp-4]`/`[ebp-8]` = locals.
5. `sub esp, 8`: allocates two dword locals; ESP → `ba70`.
6. Two `mov` stores fill the locals (no ESP change; these are memory writes, not pushes).
7. Two `mov` loads read them into EAX and EBX. Verified in gdb: EAX=100, EBX=200.
8. `mov esp, ebp`: ESP back to `ba78`, discarding locals.
9. `pop ebp`: EBP restored to 0; ESP → `ba7c`.
10. `ret`: EIP ← `0x8049005`, ESP → `ba80` (the starting value).
11. `mov eax,1 / xor ebx,ebx / int 0x80` → `exit(0)`. Note `ebx` was 200 until `xor ebx, ebx`; this matters because in the 32-bit ABI the **exit status is taken from EBX**. Without that `xor`, the exit code would be 200.

## Observations from running it

* Exit code `0`, no output.
* ESP: `ba80 → ba7c → ba78 → ba70 → ba78 → ba7c → ba80`. Down in steps of 4 / 8, back up the same, finishing where it started.
* Every number is half the 64-bit version, which shows that "stack slot size" is determined by the CPU mode, not by the instruction text.
* Memory below ESP after return still held `0xc8` and `0x64` (stale, not erased).

## Takeaways

1. The mechanism (call/push bp/frame/pop bp/ret) is identical in 32-bit and 64-bit, but the slot width and register names change.
2. In 32-bit mode `push`/`pop`/`call`/`ret` move ESP by 4, not 8.
3. The source must be assembled and linked for the mode it was written in, otherwise the stack operations are misinterpreted.
