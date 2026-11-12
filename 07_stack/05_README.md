# 05.asm - Byte Order on the Stack: Little-Endian (64-bit)

## Summary

Pushes `0x1122334455667788` and `0x1234`, dumps the raw stack bytes, then pops them back. It shows
little-endian storage: the least significant byte of a value sits at the lowest address, so the
8 bytes of the big constant appear reversed in memory.

## Build and run

```bash
nasm -f elf64 -g -F dwarf 05.asm -o 05.o
ld 05.o -o 05
./05 ; echo "EXIT=$?"
```

Real output: no output, `EXIT=0`.

Trace, with `dump_bytes = 16` to get the `x/16xb $rsp` byte dump:

```bash
gdb -q -batch -ex 'set $bits = 64' -ex 'set $dump_words = 4' \
    -ex 'set $dump_bytes = 16' -ex 'set $extra_break = 0' \
    -x trace_engine.py ./05
```

RSP at `_start` = `0x7fffffffd6f0`.

## What NASM encoded

Two lines here are worth reading closely before the trace:

```
401000  48 b8 88 77 66 55 44 33 22 11   movabs rax,0x1122334455667788
40100a  bb 34 12 00 00                  mov    ebx,0x1234
40100f  50                             push   rax
401010  53                             push   rbx
```

The `movabs` immediate is already low byte first (`88 77 66 ...`), and `mov ebx,0x1234` is a 32-bit
write (`34 12 00 00`) that zero-extends into RBX.

## Step-by-step trace

| # | Instruction | RSP before | RSP after | Delta | What happened |
|---|---|---|---|---|---|
| 1 | `movabs rax,...7788` | `d6f0` | `d6f0` | 0 | RAX = 0x1122334455667788. Needs movabs. |
| 2 | `mov ebx,0x1234` | `d6f0` | `d6f0` | 0 | RBX = 0x1234, upper half zeroed. |
| 3 | `push rax` | `d6f0` | `d6e8` | -8 | 8 bytes of RAX stored at `d6e8`, low byte first. |
| 4 | `push rbx` | `d6e8` | `d6e0` | -8 | 8 bytes of RBX stored at `d6e0`. |
| 5 | `nop` | `d6e0` | `d6e0` | 0 | Marker for the byte dump below. |
| 6 | `pop rbx` | `d6e0` | `d6e8` | +8 | RBX = 0x1234, pushed last so popped first. |
| 7 | `pop rax` | `d6e8` | `d6f0` | +8 | RAX = 0x1122334455667788. |
| 8 | `mov eax,0x3c` | `d6f0` | `d6f0` | 0 | RAX = 60. |
| 9 | `xor rdi,rdi` | `d6f0` | `d6f0` | 0 | RDI = 0. |
| - | `syscall` | `d6f0` | - | - | Not stepped: it would kill the traced process. |

Both registers end holding their original values.

## Stack memory at the nop, word view

RSP = `0x7fffffffd6e0`, `x/4gx $rsp`:

```
0x7fffffffd6e0  0x0000000000001234   <- RSP   0x1234, pushed last  (rbx)
0x7fffffffd6e8  0x1122334455667788           full 64-bit value, pushed first (rax)
0x7fffffffd6f0  0x0000000000000001           kernel's argc
0x7fffffffd6f8  0x00007fffffffda03           kernel's argv[0]
```

## Stack memory at the nop, byte view

The `x/16xb $rsp` dump the assignment asks for:

```
0x7fffffffd6e0:  0x34  0x12  0x00  0x00  0x00  0x00  0x00  0x00
0x7fffffffd6e8:  0x88  0x77  0x66  0x55  0x44  0x33  0x22  0x11
```

Read against the values, in the direction of increasing address:

| Value | Bytes as stored | Bytes as written by hand |
|---|---|---|
| `0x0000000000001234` | `34 12 00 00 00 00 00 00` | `00 00 00 00 00 00 12 34` |
| `0x1122334455667788` | `88 77 66 55 44 33 22 11` | `11 22 33 44 55 66 77 88` |

The second row is the first column reversed, which is what little-endian means. The `0x1234` slot is
8 bytes wide even though the value needs 2, because `push rbx` pushed a whole 64-bit register.

## Observations from the real run

- **Little-endian, shown from memory not asserted.** `0x88` (least significant) is at the lower
  address `d6e8`; `0x11` (most significant) is at `d6ef`. To rebuild the value, read from high
  address down to low.
- **Nothing reverses during a push.** The CPU stores a 64-bit word at the lowered address and the
  memory system puts the low byte first. The `movabs` bytes in the instruction stream are in the
  same order, which confirms it is a property of the machine.
- **Grows down, 8 bytes per slot**, -8 and +8 measured, and **LIFO holds**: 0x1234 came off first.
- **Balanced.** RSP starts and ends at `0x7fffffffd6f0`.
- **The pops do not actually swap anything.** The source pops into `rbx` then `rax`, the same
  registers the values came from, so both end unchanged. The LIFO order is visible in the memory
  dump, not in the final registers. To see it in registers you need two different destinations, as
  01.asm does with `pop rcx` / `pop rdx`.
- **push rbx pushed 8 bytes, not 4,** even though the line that filled RBX was a 32-bit move.
  Operand size comes from the `push` instruction, not from how the register was loaded.
- **Two move encodings on consecutive lines:** `movabs` (10 bytes) because the constant exceeds 32
  bits, and a 5-byte `mov ebx,imm32` because 0x1234 fits.
- **Addresses vary per machine; the byte pattern does not.** The sequence
  `34 12 00 00 00 00 00 00 | 88 77 66 55 44 33 22 11` is the same on any x86-64 machine.

