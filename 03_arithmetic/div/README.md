# DIV: Flags Analysis

For `div` (unsigned divide), **all six arithmetic flags (CF, ZF, SF, OF, PF, AF) are undefined** according to Intel's manual. `div` does not report anything useful through flags: the CPU may leave them in any state, so their values must not be relied on or given a meaning. What matters after `div` is where the quotient and remainder end up:

| Operand size       | Dividend    | Quotient | Remainder |
|--------------------|-------------|----------|-----------|
| 8-bit (`div bl`)   | `ax`        | `al`     | `ah`      |
| 16-bit (`div bx`)  | `dx:ax`     | `ax`     | `dx`      |
| 32-bit (`div ebx`) | `edx:eax`   | `eax`    | `edx`     |

If the quotient is too big for its register, or the divisor is 0, the CPU raises a divide error (exception) instead of setting a flag.

## Program 1: `div1.asm`

**Operation:** `div bl` where `ax = 100` and `bl = 7`
**Result:** `al = 14` (quotient), `ah = 2` (remainder). Check: 14 x 7 + 2 = 100.

Flags before: all arithmetic flags cleared. After: only `AF` shown set.

| Flag | Status shown in GDB | Meaning |
|------|---------------------|---------|
| CF   | Cleared             | Undefined after `div`. |
| ZF   | Cleared             | Undefined after `div`. |
| SF   | Cleared             | Undefined after `div`. |
| OF   | Cleared             | Undefined after `div`. |
| PF   | Cleared             | Undefined after `div`. |
| AF   | Set                 | Undefined after `div`. It showed as set here, but `div` performs no nibble carry, so this is not a result of the calculation. |

## Program 2: `div2.asm`

**Operation:** `div bx` where `dx:ax = 0:50000` and `bx = 300`
**Result:** `ax = 166` (quotient), `dx = 200` (remainder). Check: 166 x 300 + 200 = 50000.

Flags before: all arithmetic flags cleared. After: only `AF` shown set.

| Flag | Status shown in GDB | Meaning |
|------|---------------------|---------|
| CF   | Cleared             | Undefined after `div`. |
| ZF   | Cleared             | Undefined after `div`. |
| SF   | Cleared             | Undefined after `div`. |
| OF   | Cleared             | Undefined after `div`. |
| PF   | Cleared             | Undefined after `div`. |
| AF   | Set                 | Undefined after `div`. Same as in program 1, a leftover state, not a result. |

**Takeaway:** Unlike `add` and `sub`, `div` gives no information through flags. The result is read from the quotient and remainder registers, and any flag values seen in GDB are arbitrary.