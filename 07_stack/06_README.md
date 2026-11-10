# 06.asm — Saving and restoring registers with PUSH/POP (32-bit)

## What the program does

This is the 32-bit version of `05.asm`. It loads EAX and EBX, pushes both onto the stack, pops them back into the same registers in reverse order, and exits with the i386 `int 0x80` system call.

```asm
BITS 32
mov eax, 0x11223344   ; full 4-byte value
mov ebx, 0x1234       ; small value
push eax
push ebx
nop
pop ebx               ; reverse order of the pushes
pop eax
mov eax, 1            ; sys_exit (i386)
xor ebx, ebx          ; status 0
int 0x80
```

## Build and run

```bash
nasm -f elf32 -g -F dwarf 06.asm -o 06.o
ld -m elf_i386 06.o -o 06
./06; echo "exit status: $?"     # exit status: 0
```

In GDB:

```bash
gdb -q ./06
(gdb) starti
(gdb) stepi
(gdb) info registers eax ebx esp
(gdb) x/3wx $esp       # 4-byte words
(gdb) x/8xb $esp       # individual bytes
```

## 32-bit push and pop

* **`push src`** (32-bit): `ESP = ESP - 4`, then the 4-byte value is written to `[ESP]`.
* **`pop dst`** (32-bit): the 4-byte value at `[ESP]` goes into `dst`, then `ESP = ESP + 4`.

Each slot is 4 bytes, half the size of a 64-bit slot.

## Step-by-step trace (values from GDB)

Initial ESP = `0xffff5950` (pointing at `argc` = 1).

| # | Instruction | ESP after | Change | Top of stack | EAX | EBX |
|---|---|---|---|---|---|---|
| — | *(entry)* | `0xffff5950` | 0 | `0x00000001` | 0 | 0 |
| 1 | `mov eax, 0x11223344` | `0xffff5950` | 0 | `0x00000001` | `0x11223344` | 0 |
| 2 | `mov ebx, 0x1234` | `0xffff5950` | 0 | `0x00000001` | `0x11223344` | `0x1234` |
| 3 | `push eax` | `0xffff594c` | **−4** | `0x11223344` | same | same |
| 4 | `push ebx` | `0xffff5948` | **−4** | `0x00001234` | same | same |
| 5 | `nop` | `0xffff5948` | 0 | `0x00001234` | same | same |
| 6 | `pop ebx` | `0xffff594c` | **+4** | `0x11223344` | same | `0x1234` |
| 7 | `pop eax` | `0xffff5950` | **+4** | `0x00000001` | `0x11223344` | `0x1234` |
| 8 | `mov eax, 1` | `0xffff5950` | 0 | | 1 | `0x1234` |
| 9 | `xor ebx, ebx` | `0xffff5950` | 0 | | 1 | 0 |
| 10 | `int 0x80` | — | — | process exits | | |

### Stack after both pushes (at the `nop`)

```
0xffff5950 | 0x00000001 |  argc
0xffff594c | 0x11223344 |  saved EAX
0xffff5948 | 0x00001234 |  <- ESP   saved EBX
```

GDB confirmed this with `x/3wx $esp`:

```
0xffff5948:  0x00001234  0x11223344  0x00000001
```

### The pops

`pop ebx` reads `0x00001234` (pushed last) into EBX and moves ESP up to `0xffff594c`. `pop eax` reads `0x11223344` into EAX and moves ESP back to `0xffff5950`. Both registers are restored to the values they had before the pushes, and the stack is balanced.

### Exit

`mov eax, 1` is `exit` in the 32-bit system call table, `xor ebx, ebx` passes status 0 (the first argument goes in EBX for `int 0x80`), and `int 0x80` traps into the kernel.

## Observations from running it

* **ESP changed by 4 per push/pop,** compared with 8 in `05.asm`. After two pushes the stack was only 8 bytes deep, against 16 bytes in the 64-bit version.
* **Little-endian layout.** The 8 bytes at ESP after the pushes were:
  ```
  (gdb) x/8xb $esp
  0x...:  0x34 0x12 0x00 0x00   0x44 0x33 0x22 0x11
  ```
  The low byte of each value comes first.
* **The value is 0x11223344, not 0x1122334455667788.** A 32-bit register holds only 4 bytes, so this program uses a shorter test value than `05.asm`; a 4-byte push could not store the 8-byte one.
* **`0x1234` still occupied a full 4-byte slot** (`0x00001234`), padded with zeros.
* **EBX has two roles here.** It is the register being saved/restored, and it is also the exit-status argument for `int 0x80`. The restored value `0x1234` was then overwritten with 0 by `xor ebx, ebx`, so the exit status is 0. Without that `xor`, the program would exit with status `0x34` (52), because Linux keeps only the low 8 bits of the status.
* **Balanced.** ESP went −4, −4, +4, +4 and ended at its starting value.
