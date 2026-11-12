# 02.asm: a real stack frame (`call`, `push rbp`, local variables, `ret`), 64-bit

## What the program does

```nasm
_start:
    call work            ; call a function
    mov rax, 60          ; (after return) exit(0)
    xor rdi, rdi
    syscall

work:
    push rbp             ; save caller's frame pointer
    mov rbp, rsp         ; RBP = our frame's anchor
    sub rsp, 16          ; reserve 16 bytes for 2 local variables

    mov qword [rbp - 8], 100     ; local #1
    mov qword [rbp - 16], 200    ; local #2

    mov rax, [rbp - 8]           ; read them back
    mov rbx, [rbp - 16]
    nop

    mov rsp, rbp         ; throw away the locals
    pop rbp              ; restore caller's RBP
    ret                  ; return
```

This is the classic **function prologue / epilogue**. It shows that the stack is not only for `push`/`pop`: it is also where functions keep **local variables** and **return addresses**.

## Build and run

```
nasm -f elf64 -g 02.asm -o 02.o && ld 02.o -o 02
./02 ; echo $?        # -> 0, no output
```

## Concepts used

* `call target` does two things: **pushes the return address** (address of the next instruction) onto the stack (RSP −= 8), then jumps to `target`.
* `ret` does a **pop into RIP**: it takes the 8 bytes at the top of the stack, puts them in RIP, and RSP += 8. So `ret` only works if RSP is pointing *exactly at the return address* when it runs.
* **RBP** is a "frame pointer": a fixed reference point inside the function. RSP moves as things are pushed, but RBP stays put, so locals can be addressed as `[rbp - N]` regardless.
* Locals live at addresses **below** RBP (negative offsets), because the stack grows downward.

## Trace (gdb `stepi`; state **before** each instruction runs)

Addresses abbreviated: `c8d0` means `0x7fffffffc8d0`.

| Instruction (about to run) | RSP | RBP | Notes |
|---|---|---|---|
| `call work` (in `_start`) | `c8d0` | 0 | stack top is argc, as in program 01 |
| `push rbp` (first in `work`) | `c8c8` | 0 | `call` pushed return addr `0x401005` at `[c8c8]`, RSP −8 |
| `mov rbp, rsp` | `c8c0` | 0 | push saved old RBP (0) at `[c8c0]`, RSP −8 |
| `sub rsp, 16` | `c8c0` | `c8c0` | RBP now = RSP: the frame anchor |
| `mov qword [rbp-8], 100` | `c8b0` | `c8c0` | RSP −16: 16 bytes reserved |
| `mov qword [rbp-16], 200` | `c8b0` | `c8c0` | `[c8b8]=100` written |
| `mov rax, [rbp-8]` | `c8b0` | `c8c0` | `[c8b0]=200` written |
| `mov rbx, [rbp-16]` | `c8b0` | `c8c0` | RAX = 100 |
| `nop` | `c8b0` | `c8c0` | RBX = 200 |
| `mov rsp, rbp` | `c8b0` | `c8c0` | |
| `pop rbp` | `c8c0` | `c8c0` | locals "deallocated": RSP jumped back up 16 |
| `ret` | `c8c8` | **0** | RBP restored to caller's value (0) |
| `mov rax, 60` (back in `_start`) | `c8d0` | 0 | `ret` popped `0x401005` into RIP |

### Picture of the stack at the `nop` (the deepest point)

```
 higher addresses
   c8d0 | 0x0000000000000001 | argc  (belongs to the kernel / _start)
   c8c8 | 0x0000000000401005 | RETURN ADDRESS  (pushed by `call`; = _start+5, i.e. `mov rax,60`)
   c8c0 | 0x0000000000000000 | saved RBP       (pushed by `push rbp`)   <== RBP points here
   c8b8 | 0x0000000000000064 | local #1 = 100   = [rbp-8]
   c8b0 | 0x00000000000000c8 | local #2 = 200   = [rbp-16]               <== RSP points here
 lower addresses
```

Everything between RBP and RSP is this function's private workspace, its **stack frame**.

## Walkthrough in words, from start to end

1. **`_start` begins.** RSP = `c8d0`. Stack only has what the kernel put there.
2. **`call work`.** The CPU pushes the address of the instruction right after the call (`0x401005`) and jumps to `work`. RSP: `c8d0 → c8c8`. This is how `work` will know where to go back to.
3. **`push rbp`.** Saves the caller's RBP (0 here). RSP → `c8c0`. We have to preserve it because the caller might be using RBP for its own frame.
4. **`mov rbp, rsp`.** RBP = `c8c0`. Now RBP points at the saved-RBP slot. From now on, `[rbp]` is the saved RBP, `[rbp+8]` is the return address, and `[rbp-8]`, `[rbp-16]` are locals.
5. **`sub rsp, 16`.** Moving RSP *down* by 16 is how a function "allocates" 16 bytes of locals. No data is written; it just moves the pointer so future pushes/calls won't overwrite the locals. RSP → `c8b0`.
6. **Writes to `[rbp-8]` and `[rbp-16]`.** Ordinary memory stores. These are *not* pushes, RSP does not change. 100 lands at `c8b8`, 200 at `c8b0`.
7. **`mov rax, [rbp-8]` / `mov rbx, [rbp-16]`.** Read them back: RAX = 100, RBX = 200 (checked in gdb).
8. **`mov rsp, rbp`.** Frees all locals in one go: RSP jumps back from `c8b0` to `c8c0`. (The values 100 and 200 are still in memory, just no longer protected.)
9. **`pop rbp`.** Restores the caller's RBP and RSP → `c8c8`. Now RSP points at the return address.
10. **`ret`.** Pops the return address into RIP and RSP → `c8d0`. Execution resumes at `0x401005` in `_start`. **RSP is exactly what it was before `call`.**
11. `_start` does `exit(0)`.

## Observations from running it

* Exit code `0`, no output.
* RBX still holds 200 and RAX 100 after `ret`, because registers are not part of the frame; only RSP/RBP are restored.
* After the function returned, memory at `c8b0`/`c8b8` still contained `0xc8` / `0x64`. Freed stack memory is not wiped.
* Initially RBP is 0 (the kernel zeroes registers at start). That's why the "saved RBP" slot shows 0.
* Net effect on RSP across the whole call: **zero** (`c8d0` before, `c8d0` after).

## Why `mov rsp, rbp` matters

Without it (or an equivalent `add rsp, 16`), `pop rbp` would pop the *local variable* `200` into RBP and `ret` would then pop `100` as the return address, jumping to address `0x64`, an instant **segfault**. `ret` blindly trusts whatever is at `[RSP]`. This is also the mechanism behind stack-smashing bugs.

## Takeaways

1. `call` pushes a return address; `ret` pops it into RIP.
2. `push rbp / mov rbp, rsp` creates a stable reference; `mov rsp, rbp / pop rbp` tears it down.
3. Locals are allocated simply by subtracting from RSP and are addressed relative to RBP (negative offsets).
4. The frame must be torn down so that RSP points at the return address exactly when `ret` runs.
