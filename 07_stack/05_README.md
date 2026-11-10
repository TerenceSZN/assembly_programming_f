# 05.asm — Saving and restoring registers with PUSH/POP (64-bit)

## What the program does

Loads a full 64-bit value and a small value into RAX and RBX, pushes both, then pops them back into the **same** registers in the **reverse** order. This is the standard pattern for preserving registers: whatever you push last you must pop first. The program then exits with status 0.

```asm
mov rax, 0x1122334455667788   ; full 8-byte value
mov rbx, 0x1234               ; small value
push rax
push rbx
nop
pop rbx                        ; reverse order of the pushes
pop rax
mov rax, 60                    ; sys_exit
xor rdi, rdi
syscall
```

## Build and run

```bash
nasm -f elf64 -g -F dwarf 05.asm -o 05.o
ld 05.o -o 05
./05; echo "exit status: $?"     # exit status: 0
```

In GDB:

```bash
gdb -q ./05
(gdb) starti
(gdb) stepi
(gdb) info registers rax rbx rsp
(gdb) x/2gx $rsp       # as quadwords
(gdb) x/16xb $rsp      # as individual bytes
```

## Step-by-step trace (values from GDB)

Initial RSP = `0x7fffffff64b0` (pointing at `argc` = 1).

| # | Instruction | RSP after | Change | Top of stack | RAX | RBX |
|---|---|---|---|---|---|---|
| — | *(entry)* | `…64b0` | 0 | `0x1` | 0 | 0 |
| 1 | `mov rax, 0x1122…88` | `…64b0` | 0 | `0x1` | `0x1122334455667788` | 0 |
| 2 | `mov rbx, 0x1234` | `…64b0` | 0 | `0x1` | `0x1122334455667788` | `0x1234` |
| 3 | `push rax` | `…64a8` | **−8** | `0x1122334455667788` | same | same |
| 4 | `push rbx` | `…64a0` | **−8** | `0x0000000000001234` | same | same |
| 5 | `nop` | `…64a0` | 0 | `0x1234` | same | same |
| 6 | `pop rbx` | `…64a8` | **+8** | `0x1122334455667788` | same | `0x1234` |
| 7 | `pop rax` | `…64b0` | **+8** | `0x1` | `0x1122334455667788` | `0x1234` |
| 8–10 | exit sequence | `…64b0` | 0 | `0x1` | 60 | `0x1234` |

### 1. Loading the registers

`mov rax, 0x1122334455667788` needs a full 64-bit immediate, so NASM encodes it as the 10-byte `movabs` instruction (GDB shows `movabs $0x1122334455667788,%rax`). `mov rbx, 0x1234` fits in 32 bits, so NASM uses the shorter `mov ebx, 0x1234`, which also clears the upper half of RBX.

### 2. `push rax`

RSP decreases by 8 and all 8 bytes of RAX are stored.

```
0x7fffffff64b0 | 0x0000000000000001 |  argc
0x7fffffff64a8 | 0x1122334455667788 |  <- RSP   saved RAX
```

### 3. `push rbx`

RSP decreases by 8 again. Even though `0x1234` is only 2 significant bytes, **a 64-bit push always writes a full 8 bytes**, padded with zeros.

```
0x7fffffff64b0 | 0x0000000000000001 |  argc
0x7fffffff64a8 | 0x1122334455667788 |  saved RAX
0x7fffffff64a0 | 0x0000000000001234 |  <- RSP   saved RBX
```

### 4. `pop rbx` then `pop rax`

`pop rbx` takes the top value (`0x1234`, the last thing pushed) into RBX and raises RSP to `…64a8`. `pop rax` takes `0x1122334455667788` into RAX and raises RSP back to `…64b0`. Each register gets back exactly what it had, because the pops are in the reverse order of the pushes.

If the pops were written in the same order as the pushes (`pop rax` then `pop rbx`), the two registers would be **swapped**. That is what happened deliberately in `01.asm`, where the values ended up in different registers.

### 5. Exit

`mov rax, 60` / `xor rdi, rdi` / `syscall`, exit status 0.

## Observations from running it

* **Byte order on the stack is little-endian.** Viewing the 16 bytes at RSP after the two pushes:
  ```
  (gdb) x/16xb $rsp
  0x...a0:  0x34 0x12 0x00 0x00 0x00 0x00 0x00 0x00     <- 0x1234
  0x...a8:  0x88 0x77 0x66 0x55 0x44 0x33 0x22 0x11     <- 0x1122334455667788
  ```
  The least significant byte (`0x88`, `0x34`) is stored at the lowest address. When viewed as quadwords (`x/2gx`) GDB reassembles them into the normal-looking numbers.
* **Small values still take a full slot.** `0x1234` occupied 8 bytes of stack, the same as the full 64-bit value. Push/pop size depends on the register size (RAX/RBX = 8 bytes), not on how big the number is.
* **All 64 bits survived the round trip.** RAX was `0x1122334455667788` both before the push and after the pop, so nothing was truncated.
* **The values never changed while on the stack;** the registers also kept their values the whole time. In a real program, code between the pushes and pops would freely use RAX and RBX, and the pops would restore the originals. This is how functions preserve callee-saved registers.
* **Balanced.** RSP moved −8, −8, +8, +8 and ended where it started.
