# ADD instruction: flags analysis

## Program 1: add1.asm

Code run:

    mov al, [num1]      ; al = 120 (01111000)
    add al, [num2]      ; al = al + 10 (00001010)

Result: AL = 0x82 (10000010 in binary, 130 unsigned, -126 signed)

Flags before the add: `eflags 0x202 [ IF ]`
Flags after the add: `eflags 0xa96 [ PF AF SF IF OF ]`

| Flag | Status | Reason |
|------|--------|--------|
| CF | Clear (0) | Unsigned 120 + 10 = 130 fits in 8 bits (max 255), so there is no carry out of the top bit. |
| ZF | Clear (0) | The result 10000010 is not zero. |
| SF | Set (1) | The most significant bit of the result is 1, so it is negative when read as signed. |
| OF | Set (1) | Signed 120 + 10 = +130, which is greater than the 8-bit signed maximum of +127. Two positive numbers produced a negative-looking result, so signed overflow occurred. |
| PF | Set (1) | The low byte 10000010 has two 1 bits, which is an even count. |
| AF | Set (1) | Adding the low nibbles 1000 + 1010 = 18 carries from bit 3 into bit 4. |
| IF | Set (1) | Interrupt flag, set by the operating system. It is not affected by ADD. |


## Program 2: add2.asm

Code run (16-bit addition):

    mov ax, [num1]      ; ax = 32000 (0x7D00)
    add ax, [num2]      ; ax = ax + 500 (0x01F4)

Result: AX = 0x7EF4 (32500)

Flags before the add: `eflags 0x202 [ IF ]`
Flags after the add: `eflags 0x202 [ IF ]` (no arithmetic flags set)

| Flag | Status | Reason |
|------|--------|--------|
| CF | Clear (0) | Unsigned 32000 + 500 = 32500 fits in 16 bits (max 65535), so there is no carry out of the top bit. |
| ZF | Clear (0) | The result 0x7EF4 is not zero. |
| SF | Clear (0) | The most significant bit of 0x7EF4 (0111...) is 0, so the result is positive. |
| OF | Clear (0) | Signed 32000 + 500 = 32500 is within the 16-bit signed range (max +32767), so no signed overflow. |
| PF | Clear (0) | The low byte 0xF4 = 11110100 has five 1 bits, an odd count. |
| AF | Clear (0) | The low nibbles 0x0 + 0x4 = 0x4, so there is no carry from bit 3 into bit 4. |

