# DIV instruction: flags analysis

After DIV, all six arithmetic flags (CF, OF, SF, ZF, AF, PF) are **undefined**. The CPU does not guarantee any value for them, so whatever GDB displays is not a meaningful result of the division. The real results of DIV are the quotient and remainder registers. DIV does not report errors through flags: if the divisor is 0 or the quotient does not fit in the destination register, the CPU raises a divide error exception (#DE) instead.

## Program 1: div1.asm

Code run (8-bit unsigned divide):

    mov ax, [dividend]  ; ax = 100
    mov bl, [divisor]   ; bl = 7
    div bl              ; al = ax / bl, ah = ax % bl

Result: AL = 0x0E (14, the quotient), AH = 0x02 (2, the remainder). 100 / 7 = 14 remainder 2.
AX = 0x020E. The quotient 14 fits in AL, so no divide error occurred.

Flags before the div: `eflags 0x202 [ IF ]`
Flags after the div: `eflags 0x212 [ AF IF ]`

| Flag | Status | Reason |
|------|--------|--------|
| CF | Undefined | CF is undefined after DIV. GDB shows it clear, but this is not a result of the division. |
| OF | Undefined | OF is undefined after DIV. GDB shows it clear, but this is not a result of the division. |
| SF | Undefined | SF is undefined after DIV. GDB shows it clear, but this is not a result of the division. |
| ZF | Undefined | ZF is undefined after DIV. GDB shows it clear, but this is not a result of the division. |
| PF | Undefined | PF is undefined after DIV. GDB shows it clear, but this is not a result of the division. |
| AF | Undefined | AF is undefined after DIV. GDB shows it set (AF changed from 0 to 1 across the instruction), which is an example of why undefined flags cannot be explained from the result. |


## Program 2: div2.asm

Code run (16-bit unsigned divide, 32-bit dividend in DX:AX):

    mov ax, [dividend]  ; ax = 50000 (0xC350)
    mov dx, [highpart]  ; dx = 0
    mov bx, [divisor]   ; bx = 300 (0x012C)
    div bx              ; ax = DX:AX / bx, dx = DX:AX % bx

Result: AX = 0x00A6 (166, the quotient), DX = 0x00C8 (200, the remainder). 50000 / 300 = 166 remainder 200.
The quotient 166 fits in 16 bits, so no divide error occurred.

Flags before the div: `eflags 0x202 [ IF ]`
Flags after the div: `eflags 0x212 [ AF IF ]`

| Flag | Status | Reason |
|------|--------|--------|
| CF | Undefined | CF is undefined after DIV. GDB shows it clear, but this is not a result of the division. |
| OF | Undefined | OF is undefined after DIV. GDB shows it clear, but this is not a result of the division. |
| SF | Undefined | SF is undefined after DIV. GDB shows it clear, but this is not a result of the division. |
| ZF | Undefined | ZF is undefined after DIV. GDB shows it clear, but this is not a result of the division. |
| PF | Undefined | PF is undefined after DIV. GDB shows it clear, but this is not a result of the division. |
| AF | Undefined | AF is undefined after DIV. GDB shows it set again, as in div1, but the manual does not define it, so it cannot be explained from the result. |

