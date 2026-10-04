# MUL: Flags Analysis

For `mul` (unsigned multiply), only **CF** and **OF** have a defined meaning. Both are set when the upper half of the result is non-zero (the product did not fit in the lower half), and cleared when it did fit. The flags **ZF, SF, PF and AF are undefined** after `mul` according to Intel's manual: the CPU may leave them in any state, so their values must not be relied on or explained.

The product is twice the size of the operands: `ax` for an 8-bit multiply, `dx:ax` for 16-bit, and `edx:eax` for 32-bit.

## Program 1: `mul1.asm`

**Operation:** `mul byte [num2]` where `al = 25` and `num2 = 10`
**Result:** `ax = 0x00FA` (250), so `ah = 0`

Flags after: all arithmetic flags cleared.

| Flag | Status                    | Why |
|------|---------------------------|-----|
| CF   | Cleared                   | 250 fits in 8 bits, so the upper half (`ah`) is 0. |
| OF   | Cleared                   | Same reason: the upper half of the result is 0. |
| ZF   | Undefined (shown cleared) | Not defined by `mul`. GDB showed it cleared, but this should not be relied on. |
| SF   | Undefined (shown cleared) | Not defined by `mul`. |
| PF   | Undefined (shown cleared) | Not defined by `mul`. |
| AF   | Undefined (shown cleared) | Not defined by `mul`. |

## Program 2: `mul2.asm`

**Operation:** `mul word [num2]` where `ax = 3000` (0x0BB8) and `num2 = 200`
**Result:** `dx:ax = 0x0009:0x27C0` (600,000)

Flags after: `CF` and `OF` set.

| Flag | Status                    | Why |
|------|---------------------------|-----|
| CF   | Set                       | 600,000 does not fit in 16 bits, so the upper half (`dx = 9`) is non-zero. |
| OF   | Set                       | Same reason: the upper half of the result is non-zero. |
| ZF   | Undefined (shown cleared) | Not defined by `mul`. |
| SF   | Undefined (shown cleared) | Not defined by `mul`. |
| PF   | Undefined (shown cleared) | Not defined by `mul`. |
| AF   | Undefined (shown cleared) | Not defined by `mul`. |

**Takeaway:** For `mul`, CF and OF simply answer "did the result need the upper half?". It was no for 25 × 10 and yes for 3000 × 200.