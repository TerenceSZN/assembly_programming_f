# 01.asm — PUSH and POP in 64-bit mode (LIFO order)

## What the program does

Loads two numbers into registers, pushes both onto the stack, then pops them back into **different** registers. Because the stack is Last-In-First-Out, the values come back in reverse order. Then the program exits with status 0.

```asm
mov rax, 100      ; RAX = 0x64
mov rbx, 200      ; RBX = 0xC8
push rax          ; stack: [100]
push rbx          ; stack: [200, 100]   <- 200 on top
nop
pop rcx           ; RCX = 200  (last in, first out)
pop rdx           ; RDX = 100
nop
mov rax, 60       ; sys_exit
xor rdi, rdi      ; status 0
syscall
```

## Build and run

```bash
nasm -f elf64 -g -F dwarf 01.asm -o 01.o
ld 01.o -o 01
./01; echo "exit status: $?"     # exit status: 0
```

The program prints nothing, so the stack behaviour has to be observed in GDB:

```bash
gdb -q ./01
(gdb) starti                 # stop on the very first instruction
(gdb) stepi                  # one instruction at a time
(gdb) info registers rax rbx rcx rdx rsp
(gdb) x/4gx $rsp             # show 4 quadwords starting at the top of the stack
```

## How the stack works, step by step

Two rules explain everything in this program:

* **`push src`** (64-bit): `RSP = RSP - 8`, then the 8-byte value is written to `[RSP]`.
* **`pop dst`** (64-bit): the 8-byte value at `[RSP]` is copied into `dst`, then `RSP = RSP + 8`.

The stack grows **downwards** toward lower addresses, so pushing makes RSP smaller and popping makes it bigger.

Actual values recorded in GDB (S = initial RSP = `0x7fffffff64b0`):

| # | Instruction | RSP after | Change | Top of stack after | Registers that changed |
|---|---|---|---|---|---|
| — | *(program entry)* | `0x7fffffff64b0` | — | `0x1` (argc) | all 0 |
| 1 | `mov rax, 100` | `0x7fffffff64b0` | 0 | `0x1` | RAX = 0x64 |
| 2 | `mov rbx, 200` | `0x7fffffff64b0` | 0 | `0x1` | RBX = 0xC8 |
| 3 | `push rax` | `0x7fffffff64a8` | **−8** | `0x64` | — |
| 4 | `push rbx` | `0x7fffffff64a0` | **−8** | `0xC8` | — |
| 5 | `nop` | `0x7fffffff64a0` | 0 | `0xC8` | — |
| 6 | `pop rcx` | `0x7fffffff64a8` | **+8** | `0x64` | RCX = 0xC8 (200) |
| 7 | `pop rdx` | `0x7fffffff64b0` | **+8** | `0x1` | RDX = 0x64 (100) |
| 8 | `nop` | `0x7fffffff64b0` | 0 | `0x1` | — |
| 9–11 | exit sequence | `0x7fffffff64b0` | 0 | `0x1` | RAX = 60, RDI = 0 |

### 1. Program start

When Linux starts the program, RSP already points to a valid stack the kernel built. The first 8 bytes at `[RSP]` are `argc` (1, because we ran `./01` with no arguments), followed by the `argv` pointers and environment. The program never touches that data; it only builds on top of it (at lower addresses).

```
0x7fffffff64b0 | 0x0000000000000001 |  <- RSP (argc)
```

### 2. `mov rax, 100` and `mov rbx, 200`

These are ordinary register moves. They do not touch the stack, so RSP stays at `0x7fffffff64b0`.

### 3. `push rax`

RSP drops by 8 to `0x7fffffff64a8`, and the 8 bytes of RAX (100) are stored there.

```
0x7fffffff64b0 | 0x0000000000000001 |  argc
0x7fffffff64a8 | 0x0000000000000064 |  <- RSP   (100)
```

### 4. `push rbx`

RSP drops another 8 to `0x7fffffff64a0`, and RBX (200) is stored. 200 is now the **top** of the stack.

```
0x7fffffff64b0 | 0x0000000000000001 |  argc
0x7fffffff64a8 | 0x0000000000000064 |  100
0x7fffffff64a0 | 0x00000000000000c8 |  <- RSP   (200)
```

The stack is now 16 bytes deeper than at the start. The `nop` gives a convenient place to stop and inspect this state.

### 5. `pop rcx`

The value at `[RSP]`, 200, is copied into RCX, and RSP goes back up by 8 to `0x7fffffff64a8`. **RCX gets 200, not 100**, because 200 was pushed last.

### 6. `pop rdx`

The value now at `[RSP]`, 100, goes into RDX, and RSP returns to `0x7fffffff64b0`, exactly where it started. The stack is balanced: two pushes, two pops.

### 7. Exit

`mov rax, 60` selects the `exit` system call, `xor rdi, rdi` sets the status to 0, and `syscall` ends the process. None of these use the stack.

## Observations from running it

* **LIFO is visible in the registers.** Values went in as RAX=100 then RBX=200, and came out as RCX=200 then RDX=100.
* **RSP changes by exactly 8 per push/pop** in 64-bit mode, and only push/pop changed it. The `mov`, `nop` and `xor` instructions left RSP alone.
* **The stack was balanced.** RSP ended at the same address it started at (`0x…64b0`), which is required before returning from a function. Here it is just good practice, because `syscall exit` does not depend on RSP.
* **Pop does not erase memory.** After both pops I examined memory just *below* RSP and the old values were still there:
  ```
  (gdb) x/2gx $rsp-16
  0x...:  0x00000000000000c8  0x0000000000000064
  ```
  `pop` only moves RSP; the bytes become "free" space that the next push will overwrite.
* **Push/pop copies, it does not move.** After the pops, RAX still held 100 and RBX still held 200. Pushing a register does not clear it.
* The exact addresses (`0x7fffffff64b0`, etc.) depend on the environment (path, environment variables). The *differences* (−8, −8, +8, +8) are always the same.
