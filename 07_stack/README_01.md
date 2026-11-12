# 01.asm: `push` and `pop` (64-bit), LIFO order

## What the program does

```nasm
_start:
    mov rax, 100      ; RAX = 100
    mov rbx, 200      ; RBX = 200
    push rax          ; stack gets 100
    push rbx          ; stack gets 200
    nop               ; <-- breakpoint-style "look here" marker
    pop rcx           ; RCX = top of stack
    pop rdx           ; RDX = next top of stack
    nop
    mov rax, 60       ; exit syscall
    xor rdi, rdi      ; exit code 0
    syscall
```

It prints nothing. The point is what happens to the **stack** and **RSP**.

## Build and run

```
nasm -f elf64 -g 01.asm -o 01.o && ld 01.o -o 01
./01 ; echo $?        # -> exit code 0, no output
```

## How the stack works

* The stack is just an ordinary region of memory the kernel gives every process.
* **RSP** (the stack pointer) is a register holding the address of the **top** of the stack.
* On x86 the stack **grows toward lower addresses**. "Pushing" moves RSP *down*, "popping" moves it *up*. This surprises most people; "top of stack" means "lowest address in use".
* `push X` is exactly two micro-steps:
  1. `RSP = RSP - 8` (8 bytes in 64-bit mode)
  2. write X into memory at `[RSP]`
* `pop Y` is the reverse:
  1. `Y = [RSP]` (read 8 bytes)
  2. `RSP = RSP + 8`
* **Pop does not erase memory.** It only moves RSP. The old value is still physically there, but it's now "above the top", so the next push will overwrite it.
* The order is **LIFO** (last in, first out). Whatever was pushed last comes back first.

## When execution begins

The Linux kernel sets up the stack *before* `_start` runs. In gdb, at the first instruction:

```
RSP = 0x7fffffffc8d0
[RSP]       = 0x1        <- argc (the program was started with 1 argument: its own name)
[RSP + 8]   = 0x7fffffffcc75   <- pointer to argv[0]
```

So the stack is **not empty** at `_start`. It already holds `argc`, the `argv` pointers, etc. Our program never touches that data; it just pushes *below* it (at lower addresses).
(Exact addresses vary between machines and runs, because of environment size and ASLR. Only the *differences* matter.)

## Step-by-step trace (captured with gdb `stepi`)

The values shown are the state **after** the instruction executes.

| # | Instruction | RSP after | Stack contents (low → high address) | Registers |
|---|-------------|-----------|-------------------------------------|-----------|
| 0 | *(start of `_start`)* | `…c8d0` | `[c8d0]=1 (argc)` | all 0 |
| 1 | `mov rax, 100` | `…c8d0` | unchanged | RAX=100 |
| 2 | `mov rbx, 200` | `…c8d0` | unchanged | RBX=200 |
| 3 | `push rax` | `…c8c8` (−8) | `[c8c8]=100` , `[c8d0]=1` | unchanged |
| 4 | `push rbx` | `…c8c0` (−8) | `[c8c0]=200`, `[c8c8]=100`, `[c8d0]=1` | unchanged |
| 5 | `nop` | `…c8c0` | same | marker |
| 6 | `pop rcx` | `…c8c8` (+8) | `[c8c0]=200` (stale), `[c8c8]=100`, … | **RCX=200** |
| 7 | `pop rdx` | `…c8d0` (+8) | `[c8c0]=200`, `[c8c8]=100` (both stale) | **RDX=100** |
| 8 | `nop` | `…c8d0` | same | |
| 9-11 | `mov rax,60` / `xor rdi,rdi` / `syscall` | `…c8d0` | | exit(0) |

### Picture of the stack at the first `nop` (after both pushes)

```
 higher addresses
   0x7fffffffc8d0 | 0x0000000000000001 |  argc (kernel put it here)
   0x7fffffffc8c8 | 0x0000000000000064 |  100  <- pushed first (from RAX)
   0x7fffffffc8c0 | 0x00000000000000c8 |  200  <- pushed second (from RBX)  <== RSP points here (TOP)
 lower addresses
```

Memory dump taken by gdb at that moment: `x/2gx $rsp` → `0x7fffffffc8c0: 0xc8 0x64`. 200 is at the lower address (the top), 100 is just above it.

## Why RCX = 200 and RDX = 100 (not the other way around)

This is the whole lesson of the program. We pushed 100 then 200, so 200 is on top. `pop rcx` takes the top → **RCX = 200**. `pop rdx` then takes what is now on top → **RDX = 100**. The values come out **reversed** compared to the order they went in. That is LIFO.

To get RCX=100 and RDX=200, the pops would have to be in the opposite order (`pop rdx` first), or the pushes would have to be.

## Observations from running it

* Exit code: `0`. No output (nothing calls `write`).
* Every `push` lowered RSP by exactly **8** (`c8d0 → c8c8 → c8c0`); every `pop` raised it by exactly 8 back. After 2 pushes and 2 pops RSP is **back to its starting value** `0x…c8d0`. A balanced program leaves RSP where it found it.
* The `RAX` and `RBX` registers are **not changed** by `push` (push only *copies* the value to memory). RAX is still 100 and RBX still 200 after popping.
* After both pops, I dumped memory at `RSP-16` and the values `0xc8` and `0x64` were **still sitting there** below RSP. Pop doesn't clear memory; it just stops considering it part of the stack.

## Takeaways

1. `push` = decrement RSP, then store. `pop` = load, then increment RSP.
2. The stack grows **down**; RSP always points at the last item pushed.
3. LIFO means pops return values in the reverse order of pushes.
4. Pushes and pops must balance, otherwise RSP ends up somewhere unexpected (this matters hugely in the later programs, because `ret` pops a return address).
