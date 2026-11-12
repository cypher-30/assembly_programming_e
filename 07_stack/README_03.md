# 03.asm: the stack as a return-address keeper (`call` / `ret` only), 64-bit

## What the program does

```nasm
_start:
    mov rax, 10
    call calculate     ; jump to calculate, remembering where to come back
after_call:
    ; RAX should now be 15
    nop
    mov rax, 60
    xor rdi, rdi
    syscall

calculate:
    add rax, 5
    ret
```

There is **no `push` or `pop` written anywhere** in this program, yet the stack is still used. `call` and `ret` use it implicitly. This program isolates that one job of the stack: **remembering where to return to**.

## Build and run

```
nasm -f elf64 -g 03.asm -o 03.o && ld 03.o -o 03
./03 ; echo $?        # -> 0, no output
```

## Trace (gdb `stepi`; state **before** each instruction)

`c8d0` means `0x7fffffffc8d0`.

| Instruction about to run | RIP | RSP | Notes |
|---|---|---|---|
| `mov rax, 10` | `0x401000` | `c8d0` | `[c8d0]` = argc (kernel's data) |
| `call calculate` | `0x401005` | `c8d0` | RAX = 10 |
| `add rax, 5` | `0x401015` | **`c8c8`** | `call` pushed `0x40100a` at `[c8c8]` |
| `ret` | `0x401019` | `c8c8` | RAX = 15 |
| `nop` (`after_call`) | `0x40100a` | **`c8d0`** | `ret` popped `0x40100a` into RIP |
| `mov rax,60` / `xor` / `syscall` | … | `c8d0` | exit(0) |

### What `call` really did

`call calculate` sits at `0x401005` and is 5 bytes long, so the next instruction is at `0x40100a` (that's `after_call`). The CPU performs, in one instruction:

```
RSP = RSP - 8              ; c8d0 -> c8c8
[RSP] = 0x40100a           ; push the return address
RIP = address of calculate ; 0x401015
```

Stack right after the `call` (and during `calculate`):

```
 higher
   c8d0 | 0x0000000000000001 | argc
   c8c8 | 0x000000000040100a | return address = after_call   <== RSP (top)
 lower
```

### What `ret` really did

```
RIP = [RSP]                ; RIP = 0x40100a
RSP = RSP + 8              ; c8c8 -> c8d0
```

So `ret` is effectively `pop rip`. Execution continues at `after_call`, and **RSP is back to `c8d0`**, exactly as before the call.

## The key idea: why the stack and not a register?

`calculate` could be called from anywhere, and it could call other functions. The CPU needs to remember *one return address per active call*, and in the right order (the most recent call returns first). That's exactly LIFO, which is what a stack is. If there were a single "return register", a nested call would overwrite it.

## Observations from running it

* Exit code `0`, no output.
* In gdb: RAX = 10 before the call, **RAX = 15** after `add rax, 5`, and it's still 15 at `after_call` (the comment in the source, "RAX should now be 15", is confirmed). Registers are shared between caller and callee, so no stack is needed to pass this value back.
* RSP went `c8d0 → c8c8 → c8d0`. One `call` = −8, one `ret` = +8.
* The return address `0x40100a` sat at `[c8c8]` for the duration of `calculate`. In the dumps from after `ret`, that value is still in memory at `c8c8` (stale, below RSP).

## What would break if RSP were unbalanced

If `calculate` did a `push rax` and forgot to `pop`, then `ret` would pop **RAX's value** (e.g. 15) as the return address and jump to address `0xf`, causing a segfault. `ret` can't tell a return address from any other 8 bytes; it trusts RSP. Hence the rule: **a function must leave RSP exactly as it found it** before executing `ret`.

## Takeaways

1. `call X` ≡ `push <next instruction address>` + `jmp X`.
2. `ret` ≡ `pop rip`.
3. Because the stack is LIFO, nested calls return in the right order automatically.
4. This function needed no locals, so it needs no frame (compare with 02.asm).
