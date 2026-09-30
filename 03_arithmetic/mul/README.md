# MUL instruction: flags analysis

For MUL, only CF and OF are defined. They are set when the upper half of the result is not zero (the result does not fit in the lower half). SF, ZF, AF and PF are **undefined** after MUL, so the values GDB shows for them are not meaningful.

## Program 1: mul1.asm

Code run (8-bit unsigned multiply):

    mov al, [num1]      ; al = 25
    mul byte [num2]     ; ax = al * 10

Result: AX = 0x00FA (250). AH = 0x00, AL = 0xFA

Flags before the mul: `eflags 0x202 [ IF ]`
Flags after the mul: `eflags 0x202 [ IF ]`

| Flag | Status | Reason |
|------|--------|--------|
| CF | Clear (0) | The upper half of the result (AH) is 0, so 25 x 10 = 250 fits in 8 bits. |
| OF | Clear (0) | OF follows CF for MUL. AH is 0, so there is no overflow into the upper half. |
| SF | Undefined | SF is undefined after MUL. GDB shows it clear, but this is not a result of the multiplication. |
| ZF | Undefined | ZF is undefined after MUL. GDB shows it clear, but this is not a result of the multiplication. |
| AF | Undefined | AF is undefined after MUL. GDB shows it clear, but this is not a result of the multiplication. |
| PF | Undefined | PF is undefined after MUL. GDB shows it clear, but this is not a result of the multiplication. |


## Program 2: mul2.asm

Code run (16-bit unsigned multiply):

    mov ax, [num1]      ; ax = 3000 (0x0BB8)
    mul word [num2]     ; DX:AX = ax * 200 (0x00C8)

Result: DX:AX = 0x000927C0 (600000). DX = 0x0009, AX = 0x27C0

Flags before the mul: `eflags 0x202 [ IF ]`
Flags after the mul: `eflags 0xa03 [ CF IF OF ]`

| Flag | Status | Reason |
|------|--------|--------|
| CF | Set (1) | The upper half of the result (DX = 0x0009) is not zero, so 3000 x 200 = 600000 does not fit in 16 bits (max 65535). |
| OF | Set (1) | OF follows CF for MUL. DX is not zero, so the result overflowed into the upper half. |
| SF | Undefined | SF is undefined after MUL. GDB shows it clear, but this is not a result of the multiplication. |
| ZF | Undefined | ZF is undefined after MUL. GDB shows it clear, but this is not a result of the multiplication. |
| AF | Undefined | AF is undefined after MUL. GDB shows it clear, but this is not a result of the multiplication. |
| PF | Undefined | PF is undefined after MUL. GDB shows it clear, but this is not a result of the multiplication. |

