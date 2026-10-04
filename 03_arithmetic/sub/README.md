# SUB: Flags Analysis

For subtraction, CF acts as a **borrow** flag: it is set when the unsigned subtraction needs to borrow (first number smaller than second).

## Program 1: `sub1.asm`

**Operation:** `sub al, [num2]` where `al = 50` (0x32) and `num2 = 80` (0x50)
**Result:** `al = 0xE2` (binary `11100010`, which is -30 as a signed number)

Flags before: all arithmetic flags cleared. After: `CF PF SF` set.

| Flag | Status  | Why |
|------|-------- |-----|
| CF   | Set     | 50 < 80 as unsigned numbers, so the subtraction needed a borrow. |
| ZF   | Cleared | The result is not zero. |
| SF   | Set     | The top bit of `11100010` is 1. |
| OF   | Cleared | As signed numbers, 50 - 80 = -30, which fits in 8 bits (-128 to +127). |
| PF   | Set     | The low byte `11100010` has four 1-bits, an even count. |
| AF   | Cleared | Low nibble 2 - 0 does not need a borrow from bit 4. |

**Takeaway:** CF is set but OF is cleared. Unsigned, the result went below zero. Signed, -30 is perfectly valid.

## Program 2: `sub3.asm`

This program runs two instructions: `sub ax, [num2]` with `ax = 0` and `num2 = 1`, then `sbb ax, 0`.

### Step 1: `sub ax, 1`
**Result:** `ax = 0xFFFF`

| Flag | Status  | Why |
|------|-------- |-----|
| CF   | Set     | 0 < 1 as unsigned numbers, so a borrow was needed. |
| ZF   | Cleared | The result is not zero. |
| SF   | Set     | The top bit of `0xFFFF` is 1. |
| OF   | Cleared | As signed numbers, 0 - 1 = -1, which is valid. |
| PF   | Set     | The low byte `0xFF` has eight 1-bits, an even count. |
| AF   | Set     | Low nibble 0 - 1 needs a borrow from bit 4. |

### Step 2: `sbb ax, 0`
**Result:** `ax = 0xFFFE` (`0xFFFF - 0 - CF`, with CF = 1 from step 1)

| Flag | Status  | Why |
|------|-------- |-----|
| CF   | Cleared | `0xFFFF` is large enough to subtract 1, so no borrow. The earlier borrow was consumed as an input. |
| ZF   | Cleared | The result is not zero. |
| SF   | Set     | The top bit of `0xFFFE` is 1. |
| OF   | Cleared | As signed numbers, -1 - 1 = -2, which is valid. |
| PF   | Cleared | The low byte `0xFE` has seven 1-bits, an odd count. |
| AF   | Cleared | Low nibble F - 0 - 1 = E, no borrow from bit 4. |

**Takeaway:** `sbb` passes a borrow from one subtraction into the next, which is how the CPU subtracts numbers larger than a register.