# 04.asm — A function with a stack frame (32-bit)

## What the program does

This is the 32-bit version of `02.asm`. `_start` calls `work`, which builds a stack frame with EBP, reserves 8 bytes for two 4-byte locals, stores 100 and 200 in them, loads them into EAX and EBX, tears the frame down and returns. `_start` then exits using the 32-bit Linux system call interface (`int 0x80`).

```asm
BITS 32
_start:
    call work
    mov eax, 1          ; sys_exit (i386 number)
    xor ebx, ebx        ; status 0
    int 0x80

work:
    push ebp            ; save caller's EBP          (prologue)
    mov ebp, esp        ; EBP = base of this frame
    sub esp, 8          ; reserve 8 bytes for locals

    mov dword [ebp - 4], 100
    mov dword [ebp - 8], 200

    mov eax, [ebp - 4]  ; EAX = 100
    mov ebx, [ebp - 8]  ; EBX = 200
    nop

    mov esp, ebp        ; free the locals            (epilogue)
    pop ebp             ; restore caller's EBP
    ret
```

## Build and run

It must be assembled and linked as a 32-bit program:

```bash
nasm -f elf32 -g -F dwarf 04.asm -o 04.o
ld -m elf_i386 04.o -o 04
./04; echo "exit status: $?"     # exit status: 0
```

In GDB (note the 32-bit register names and `wx` = 4-byte words):

```bash
gdb -q ./04
(gdb) starti
(gdb) stepi
(gdb) info registers esp ebp eax ebx eip
(gdb) x/4wx $esp
```

## Differences from the 64-bit version

| | 02.asm (64-bit) | 04.asm (32-bit) |
|---|---|---|
| Stack pointer / base pointer | RSP / RBP | ESP / EBP |
| Size of one push/pop/return address | 8 bytes | **4 bytes** |
| Size of each local | `qword` (8) | `dword` (4) |
| Locals at | `[rbp-8]`, `[rbp-16]` | `[ebp-4]`, `[ebp-8]` |
| Space reserved | `sub rsp, 16` | `sub esp, 8` |
| Exit | `rax=60`, `syscall` | `eax=1`, `int 0x80` |

The logic of the stack frame is identical; only the slot size halves.

## Step-by-step trace (values from GDB)

Initial ESP = `0xffff5950` (pointing at `argc` = 1). EBP starts at 0.

| # | Instruction | ESP after | EBP after | Change from start | Notes |
|---|---|---|---|---|---|
| — | *(entry)* | `0xffff5950` | 0 | 0 | |
| 1 | `call work` | `0xffff594c` | 0 | **−4** | `[0xffff594c]` = `0x08049005` (return address) |
| 2 | `push ebp` | `0xffff5948` | 0 | **−8** | `[0xffff5948]` = 0 (old EBP) |
| 3 | `mov ebp, esp` | `0xffff5948` | `0xffff5948` | −8 | frame base set |
| 4 | `sub esp, 8` | `0xffff5940` | `0xffff5948` | **−16** | 8 bytes reserved |
| 5 | `mov [ebp-4], 100` | `0xffff5940` | `0xffff5948` | −16 | `[0xffff5944]` = 0x64 |
| 6 | `mov [ebp-8], 200` | `0xffff5940` | `0xffff5948` | −16 | `[0xffff5940]` = 0xC8 |
| 7 | `mov eax, [ebp-4]` | `0xffff5940` | `0xffff5948` | −16 | EAX = 0x64 |
| 8 | `mov ebx, [ebp-8]` | `0xffff5940` | `0xffff5948` | −16 | EBX = 0xC8 |
| 9 | `nop` | `0xffff5940` | `0xffff5948` | −16 | |
| 10 | `mov esp, ebp` | `0xffff5948` | `0xffff5948` | −8 | locals discarded |
| 11 | `pop ebp` | `0xffff594c` | 0 | −4 | EBP restored |
| 12 | `ret` | `0xffff5950` | 0 | 0 | EIP = `0x08049005` |
| 13–15 | exit sequence | `0xffff5950` | 0 | 0 | EAX = 1, EBX = 0 |

### 1. `call work`

ESP drops by **4** (not 8), and the 4-byte return address `0x08049005` (the `mov eax, 1` after the call) is stored on top. EIP jumps to `work`.

```
0xffff5950 | 0x00000001 |  argc
0xffff594c | 0x08049005 |  <- ESP  return address
```

### 2. Prologue: `push ebp` / `mov ebp, esp`

The caller's EBP (0) is saved, and EBP is set to point at that slot. EBP is now the fixed reference for the frame.

### 3. `sub esp, 8`

ESP moves 8 bytes lower to `0xffff5940`, reserving room for two dwords.

### 4. Locals

```
0xffff594c | 0x08049005 |  return address  [ebp + 4]
0xffff5948 | 0x00000000 |  saved EBP       [ebp]      <- EBP
0xffff5944 | 0x00000064 |  local 1 = 100   [ebp - 4]
0xffff5940 | 0x000000c8 |  local 2 = 200   [ebp - 8]  <- ESP
```

GDB's `x/4wx $esp` at the `nop` printed exactly this:

```
0xffff5940:  0x000000c8  0x00000064  0x00000000  0x08049005
```

Reading left to right goes **up** in memory: local 2, local 1, saved EBP, return address.

### 5. Epilogue and return

`mov esp, ebp` discards both locals at once (ESP back to `0xffff5948`). `pop ebp` restores EBP to 0 (ESP → `0xffff594c`). `ret` pops `0x08049005` into EIP (ESP → `0xffff5950`), returning to `_start`.

### 6. Exit

`mov eax, 1` (exit in the i386 system call table), `xor ebx, ebx` (status 0) and `int 0x80` end the process. Note that this overwrites the EBX value 200 that `work` loaded.

## Observations from running it

* **Every stack step was 4 bytes**, including the return address pushed by `call`. In 64-bit `02.asm` the same steps were 8 bytes.
* **Deepest point: 16 bytes below the start** = 4 (return address) + 4 (saved EBP) + 8 (locals), exactly half of the 32 bytes seen in `02.asm`.
* **The return address sits at `[ebp+4]`** in 32-bit code (it was `[rbp+8]` in 64-bit), because the saved EBP below it is only 4 bytes.
* **Stack addresses are much lower** (`0xffff5950` vs `0x7fffffff64b0`) because a 32-bit process only has a 4 GB address space.
* **It must be linked with `-m elf_i386`.** Assembling a `BITS 32` file with `-f elf64` would not give a working 32-bit program, and `int 0x80` uses different system call numbers (exit = 1) than the 64-bit `syscall` instruction (exit = 60).
* **Balanced.** ESP and EBP returned to their starting values before the exit.
