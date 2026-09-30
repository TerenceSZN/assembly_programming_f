# SUB instruction: flags analysis

## Program 1: sub1.asm

Code run (8-bit subtraction):

    mov al, [num1]      ; al = 50 (00110010)
    sub al, [num2]      ; al = al - 80 (01010000)

Result: AL = 0xE2 (11100010 in binary, 226 unsigned, -30 signed)

Flags before the sub: `eflags 0x202 [ IF ]`
Flags after the sub: `eflags 0x287 [ CF PF SF IF ]`

| Flag | Status | Reason |
|------|--------|--------|
| CF | Set (1) | Unsigned 50 is smaller than 80, so a borrow was needed. For SUB, CF indicates a borrow. |
| ZF | Clear (0) | The result 11100010 is not zero. |
| SF | Set (1) | The most significant bit of the result is 1, so it is negative when read as signed (-30). |
| OF | Clear (0) | Signed 50 - 80 = -30 is within the 8-bit signed range (-128 to +127), so no signed overflow. |
| PF | Set (1) | The low byte 11100010 has four 1 bits, an even count. |
| AF | Clear (0) | The low nibbles 0x2 - 0x0 = 0x2 need no borrow from bit 4. |


## Program 2: sub2.asm

Code run (16-bit subtraction):

    mov ax, [num1]      ; ax = 1000 (0x03E8)
    sub ax, [num2]      ; ax = ax - 2000 (0x07D0)

Result: AX = 0xFC18 (64536 unsigned, -1000 signed)

Flags before the sub: `eflags 0x202 [ IF ]`
Flags after the sub: `eflags 0x287 [ CF PF SF IF ]`

| Flag | Status | Reason |
|------|--------|--------|
| CF | Set (1) | Unsigned 1000 is smaller than 2000, so a borrow was needed. For SUB, CF indicates a borrow. |
| ZF | Clear (0) | The result 0xFC18 is not zero. |
| SF | Set (1) | The most significant bit of 0xFC18 is 1, so the result is negative when read as signed (-1000). |
| OF | Clear (0) | Signed 1000 - 2000 = -1000 is within the 16-bit signed range (-32768 to +32767), so no signed overflow. |
| PF | Set (1) | The low byte 0x18 = 00011000 has two 1 bits, an even count. |
| AF | Clear (0) | The low nibbles 0x8 - 0x0 = 0x8 need no borrow from bit 4. |

