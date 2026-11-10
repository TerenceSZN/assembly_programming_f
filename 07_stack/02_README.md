# 02.asm — A function with a stack frame (64-bit)

## What the program does

`_start` calls a function `work`. Inside `work` the program builds a **stack frame**: it saves the old RBP, makes RBP the frame base, reserves 16 bytes for two local variables, stores 100 and 200 in them, reads them back into RAX and RBX, then tears the frame down and returns. `_start` then exits with status 0.

```asm
_start:
    call work            ; push return address, jump to work
    mov rax, 60          ; sys_exit
    xor rdi, rdi
    syscall

work:
    push rbp             ; save caller's RBP          (prologue)
    mov rbp, rsp         ; RBP = base of this frame
    sub rsp, 16          ; reserve 16 bytes for locals

    mov qword [rbp - 8], 100    ; local #1
    mov qword [rbp - 16], 200   ; local #2

    mov rax, [rbp - 8]          ; RAX = 100
    mov rbx, [rbp - 16]         ; RBX = 200
    nop

    mov rsp, rbp         ; free the locals            (epilogue)
    pop rbp              ; restore caller's RBP
    ret                  ; pop return address, jump back
```

## Build and run

```bash
nasm -f elf64 -g -F dwarf 02.asm -o 02.o
ld 02.o -o 02
./02; echo "exit status: $?"     # exit status: 0
```

Inspect it in GDB:

```bash
gdb -q ./02
(gdb) starti
(gdb) stepi                         # repeat
(gdb) info registers rsp rbp rax rbx rip
(gdb) x/4gx $rsp
```

## The instructions that change RSP here

| Instruction | Effect on RSP | What is written/read |
|---|---|---|
| `call work` | −8 | pushes the return address (address of the next instruction) |
| `push rbp` | −8 | saves the caller's RBP |
| `sub rsp, 16` | −16 | nothing written; space is just reserved |
| `mov rsp, rbp` | jumps back up to RBP | locals are discarded |
| `pop rbp` | +8 | restores the saved RBP |
| `ret` | +8 | pops the return address into RIP |

The `mov [rbp-…]` instructions **do not move RSP**. They write into space that was already reserved.

## Step-by-step trace (values from GDB)

Initial RSP = `0x7fffffff64b0` (it points at `argc` = 1). RBP starts at 0.

| # | Instruction | RSP after | RBP after | Change from start | Notes |
|---|---|---|---|---|---|
| — | *(entry)* | `…64b0` | 0 | 0 | |
| 1 | `call work` | `…64a8` | 0 | −8 | `[…64a8]` = `0x401005` (return address) |
| 2 | `push rbp` | `…64a0` | 0 | −16 | `[…64a0]` = 0 (old RBP) |
| 3 | `mov rbp, rsp` | `…64a0` | `…64a0` | −16 | frame base set |
| 4 | `sub rsp, 16` | `…6490` | `…64a0` | −32 | 16 bytes reserved |
| 5 | `mov [rbp-8], 100` | `…6490` | `…64a0` | −32 | `[…6498]` = 0x64 |
| 6 | `mov [rbp-16], 200` | `…6490` | `…64a0` | −32 | `[…6490]` = 0xC8 |
| 7 | `mov rax, [rbp-8]` | `…6490` | `…64a0` | −32 | RAX = 0x64 |
| 8 | `mov rbx, [rbp-16]` | `…6490` | `…64a0` | −32 | RBX = 0xC8 |
| 9 | `nop` | `…6490` | `…64a0` | −32 | |
| 10 | `mov rsp, rbp` | `…64a0` | `…64a0` | −16 | locals discarded |
| 11 | `pop rbp` | `…64a8` | 0 | −8 | RBP restored |
| 12 | `ret` | `…64b0` | 0 | 0 | RIP = `0x401005` |
| 13–15 | exit sequence | `…64b0` | 0 | 0 | RAX = 60, RDI = 0 |

### 1. `call work`

`call` does two things at once: it **pushes the address of the next instruction** (`mov rax, 60` at `0x401005`) and jumps to `work` at `0x40100f`. RSP drops by 8.

```
0x7fffffff64b0 | 0x0000000000000001 |  argc
0x7fffffff64a8 | 0x0000000000401005 |  <- RSP  return address
```

### 2. Prologue: `push rbp` / `mov rbp, rsp`

The caller's RBP (0, since `_start` never set it) is saved on the stack, then RBP is pointed at that saved slot. From now on RBP is a fixed reference point for this function, even if RSP moves.

```
0x7fffffff64a8 | 0x0000000000401005 |  return address   = [rbp + 8]
0x7fffffff64a0 | 0x0000000000000000 |  saved RBP        = [rbp]      <- RBP, RSP
```

### 3. `sub rsp, 16`

RSP moves down 16 bytes to `0x7fffffff6490`. This **allocates** space for two 8-byte locals without writing anything. In this run the reserved bytes happened to be 0, but in general they hold leftover garbage until written.

### 4. Writing and reading the locals

The locals are addressed **relative to RBP**, at negative offsets:

```
0x7fffffff64a8 | 0x0000000000401005 |  return address   [rbp + 8]
0x7fffffff64a0 | 0x0000000000000000 |  saved RBP        [rbp]       <- RBP
0x7fffffff6498 | 0x0000000000000064 |  local 1 = 100    [rbp - 8]
0x7fffffff6490 | 0x00000000000000c8 |  local 2 = 200    [rbp - 16]  <- RSP
```

`mov rax, [rbp-8]` and `mov rbx, [rbp-16]` then load 100 and 200 into RAX and RBX. None of these instructions change RSP.

### 5. Epilogue: `mov rsp, rbp` / `pop rbp`

`mov rsp, rbp` throws away all the locals in one step: RSP jumps back to `0x…64a0`, the saved-RBP slot. `pop rbp` restores the caller's RBP (0) and moves RSP to `0x…64a8`, which is the return address.

### 6. `ret`

`ret` pops the return address `0x401005` into RIP and adds 8 to RSP, so execution continues at `mov rax, 60` in `_start`, and RSP is back to `0x…64b0`, the value it had before the call.

### 7. Exit

`mov rax, 60` / `xor rdi, rdi` / `syscall` terminate the process with status 0.

## Observations from running it

* **`call` and `ret` use the stack.** After `call`, the top of the stack held `0x401005`, the address of the instruction right after the `call`. `ret` popped it into RIP, and GDB showed execution landing back on line 6 of `_start`.
* **The deepest point was 32 bytes below the start:** 8 (return address) + 8 (saved RBP) + 16 (locals).
* **RBP stays fixed while RSP moves.** Throughout the function body RBP was `0x…64a0`, so `[rbp-8]` and `[rbp-16]` always meant the same two locals.
* **Locals live below RBP, the return address lives above it** (`[rbp+8]`). Writing past the locals upward would overwrite the saved RBP and the return address, which is how a stack buffer overflow takes over a program.
* **Alignment.** On entry to `work`, RSP was `…64a8` (8 bytes off a 16-byte boundary, because `call` pushed 8 bytes). After `push rbp` and `sub rsp, 16`, RSP was `…6490`, which is 16-byte aligned. This is why the System V x86-64 convention pairs `push rbp` with a multiple-of-16 `sub`.
* **The frame was fully undone.** After `ret`, RSP was back to the starting value and RBP was back to 0; RAX=100 and RBX=200 survived because they are registers, not stack memory.
