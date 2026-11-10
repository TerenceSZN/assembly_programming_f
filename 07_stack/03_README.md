# 03.asm — How CALL and RET use the stack (64-bit)

## What the program does

Puts 10 in RAX and calls a tiny function `calculate` that adds 5. The result (15) is passed back in RAX. Control returns to the label `after_call`, and the program exits with status 0.

```asm
_start:
    mov rax, 10
    call calculate       ; push return address, jump

after_call:
    nop                  ; RAX should now be 15
    mov rax, 60          ; sys_exit
    xor rdi, rdi
    syscall

calculate:
    add rax, 5           ; RAX = 15
    ret                  ; pop return address, jump back
```

This program does **not** use `push` or `pop` explicitly. The only stack use is the hidden push done by `call` and the hidden pop done by `ret`.

## Build and run

```bash
nasm -f elf64 -g -F dwarf 03.asm -o 03.o
ld 03.o -o 03
./03; echo "exit status: $?"     # exit status: 0
```

In GDB:

```bash
gdb -q ./03
(gdb) starti
(gdb) stepi
(gdb) info registers rax rsp rip
(gdb) x/2gx $rsp
```

## What CALL and RET actually do

* `call calculate` behaves like
  ```
  push <address of the instruction after the call>   ; RSP -= 8
  jmp  calculate
  ```
* `ret` behaves like
  ```
  pop  rip                                           ; RSP += 8
  ```

The disassembly shows where the return address comes from:

```
401005:  e8 0b 00 00 00   call 401015 <calculate>
40100a:  90               nop          <after_call>
```

The `call` instruction is 5 bytes long and starts at `0x401005`, so the next instruction is at `0x40100a`. That is exactly the value `call` pushes and exactly where the label `after_call` is.

## Step-by-step trace (values from GDB)

Initial RSP = `0x7fffffff64b0` (pointing at `argc` = 1).

| # | Instruction | RSP after | Change | Top of stack | RAX | RIP after |
|---|---|---|---|---|---|---|
| — | *(entry)* | `…64b0` | 0 | `0x1` (argc) | 0 | `0x401000` |
| 1 | `mov rax, 10` | `…64b0` | 0 | `0x1` | 0xA | `0x401005` |
| 2 | `call calculate` | `…64a8` | **−8** | `0x40100a` | 0xA | `0x401015` |
| 3 | `add rax, 5` | `…64a8` | 0 | `0x40100a` | **0xF** | `0x401019` |
| 4 | `ret` | `…64b0` | **+8** | `0x1` | 0xF | `0x40100a` |
| 5 | `nop` (after_call) | `…64b0` | 0 | `0x1` | 0xF | |
| 6–8 | exit sequence | `…64b0` | 0 | `0x1` | 60 | |

### 1. `mov rax, 10`

A plain register load. The stack is untouched.

### 2. `call calculate`

RSP decreases by 8, and the return address `0x40100a` is stored at the new top. RIP jumps to `calculate`.

```
0x7fffffff64b0 | 0x0000000000000001 |  argc
0x7fffffff64a8 | 0x000000000040100a |  <- RSP   return address (after_call)
```

### 3. `add rax, 5`

RAX becomes 15 (0xF). `calculate` never touches RSP, so the return address stays on top of the stack. This matters: `ret` will blindly pop whatever is on top, so a function must leave RSP where it found it.

### 4. `ret`

`ret` pops `0x40100a` into RIP and adds 8 to RSP. Execution continues at `after_call`, and RSP is back to `0x7fffffff64b0`.

### 5. Exit

At `after_call`, RAX = 15, confirming the comment in the source. The exit sequence then overwrites RAX with 60 and calls `exit(0)`.

## Observations from running it

* **The return address on the stack was `0x40100a`**, the address of `after_call`, not the address of `call` itself.
* **The result came back in a register, not on the stack.** RAX went 10 → 15 inside `calculate` and was still 15 after `ret`. RAX is the conventional return-value register on x86-64.
* **The stack was only one slot deep.** Maximum depth was 8 bytes (just the return address), because `calculate` has no prologue, no locals and no pushes. It is a *leaf* function that needs no frame.
* **`ret` depends entirely on RSP.** If `calculate` had pushed something and not popped it, `ret` would have jumped to that value instead of `after_call`, almost certainly crashing the program.
* **Balanced.** RSP ended exactly at its starting value.
* **The exit status is 0, not 15.** The program exits with `rdi = 0`, so `echo $?` prints 0. The value 15 can only be seen in the debugger (`info registers rax` at `after_call`).
