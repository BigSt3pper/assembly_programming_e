# ADD: Flags Analysis

## Program 1: `add1.asm`

**Operation:** `add al, [num2]` where `al = 120` (0x78) and `num2 = 10` (0x0A)
**Result:** `al = 130` (0x82, binary `10000010`)

Flags before: all arithmetic flags cleared. After: `PF AF SF OF` set.

| Flag | Status  | Why |
|------|-------- |-----|
| CF   | Cleared  | 130 fits in 8 bits unsigned (max 255), so there is no carry out of bit 7. |
| ZF   | Cleared  | The result is not zero. |
| SF   | Set      | The top bit of the result (`10000010`) is 1. |
| OF   | Set      | As signed numbers, 120 + 10 = 130 exceeds the 8-bit signed max of +127. Two positives gave a negative (-126). |
| PF   | Set      | The low byte `10000010` has two 1-bits, an even count. |
| AF   | Set      | Low nibble 8 + 10 = 18 exceeds 15, so there was a carry from bit 3 into bit 4. |

**Takeaway:** CF is cleared but OF is set. The result is valid unsigned but overflows signed.



## Program 2: `add3.asm`

This program runs two instructions: `add ax, [num2]` with `ax = 0xFFFF` and `num2 = 1`, then `adc ax, 0`.

### Step 1: `add ax, 1`
**Result:** `ax = 0x0000` (the true sum 65536 does not fit in 16 bits)

| Flag | Status  | Why |
|------|-------- |-----|
| CF   | Set     | The true result 65536 needs 17 bits, so there is a carry out of bit 15. |
| ZF   | Set     | The 16-bit result is zero. |
| SF   | Cleared | The top bit of the result is 0. |
| OF   | Cleared | As signed numbers, 0xFFFF is -1 and -1 + 1 = 0, which is valid. |
| PF   | Set     | The low byte `00000000` has zero 1-bits, an even count. |
| AF   | Set     | Low nibble F + 1 = 16, a carry from bit 3 into bit 4. |

### Step 2: `adc ax, 0`
**Result:** `ax = 0x0001` (0 + 0 + the carry from step 1)

| Flag | Status  | Why |
|------|-------- |-----|
| CF   | Cleared | The result is small, so no carry out. The carry from step 1 was consumed as an input. |
| ZF   | Cleared | The result is 1, not zero. |
| SF   | Cleared | The top bit of the result is 0. |
| OF   | Cleared | No signed overflow, 0 + 0 + 1 = 1. |
| PF   | Cleared | The low byte `00000001` has one 1-bit, an odd count. |
| AF   | Cleared | No carry from bit 3 into bit 4. |

**Takeaway:** `adc` shows how a carry from one addition is passed into the next one. This is how the CPU adds numbers bigger than its register size.