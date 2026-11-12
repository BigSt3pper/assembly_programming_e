# 06.asm - Little-Endian Push and Pop in 32-bit Mode (BITS 32, int 0x80)

## Summary

The 32-bit counterpart of 05.asm: puts `0x11223344` in EAX and `0x1234` in EBX, pushes both,
pops them back and exits with `int 0x80`. Next to 05.asm it shows how byte order and slot width
differ between 32-bit and 64-bit code.

## Build and run

```bash
nasm -f elf32 -g -F dwarf 06.asm -o 06.o
ld -m elf_i386 06.o -o 06
./06 ; echo "EXIT=$?"
```

Real output: no output, `EXIT=0`.

Trace:

```bash
gdb -q -batch -ex 'set $bits = 32' -ex 'set $dump_words = 4' \
    -ex 'set $dump_bytes = 0' -ex 'set $extra_break = 0' \
    -x trace_engine.py ./06
```

Adding `-ex 'set $dump_bytes = 16'` would print a byte dump for this program too; the assignment
only asks for that in 05.asm.

ESP at `_start` = `0xffffc860`.

## Step-by-step trace

| # | Instruction | ESP before | ESP after | Delta | What happened |
|---|---|---|---|---|---|
| 1 | `mov eax,0x11223344` | `c860` | `c860` | 0 | EAX = 287454020. No stack use. |
| 2 | `mov ebx,0x1234` | `c860` | `c860` | 0 | EBX = 4660. No stack use. |
| 3 | `push eax` | `c860` | `c85c` | -4 | 4 bytes stored at `c85c`, low byte (0x44) first. |
| 4 | `push ebx` | `c85c` | `c858` | -4 | 4 bytes of 0x00001234 stored at `c858`. |
| 5 | `nop` | `c858` | `c858` | 0 | Marker for the memory dump below. |
| 6 | `pop ebx` | `c858` | `c85c` | +4 | EBX = 0x1234, pushed last so popped first. |
| 7 | `pop eax` | `c85c` | `c860` | +4 | EAX = 0x11223344. |
| 8 | `mov eax,0x1` | `c860` | `c860` | 0 | EAX = 1 = exit syscall number (32-bit). |
| 9 | `xor ebx,ebx` | `c860` | `c860` | 0 | EBX = 0 = exit status (32-bit). |
| - | `int 0x80` | `c860` | - | - | Not stepped: it would kill the traced process. |

All addresses share the prefix `0xffff`.

## Stack memory at the nop

ESP = `0xffffc858`, `x/4wx $esp`:

```
0xffffc858  0x00001234   <- ESP   0x1234, pushed last  (ebx)
0xffffc85c  0x11223344           32-bit value, pushed first (eax)
0xffffc860  0x00000001           kernel's argc
0xffffc864  0xffffca03           kernel's argv[0]
```

The same bytes, grouped one byte at a time:

```
0xffffc858:  0x34  0x12  0x00  0x00     0x00001234, low byte first
0xffffc85c:  0x44  0x33  0x22  0x11     0x11223344, low byte first
```

That byte view is the word dump above re-grouped, not raw gdb output.

## 06.asm vs 05.asm

| Item | 05.asm (64-bit) | 06.asm (32-bit) |
|---|---|---|
| big value | `0x1122334455667788` (movabs) | `0x11223344` (plain mov) |
| push width | 8 bytes | 4 bytes |
| bytes of big value | `88 77 66 55 44 33 22 11` | `44 33 22 11` |
| RSP/ESP delta per push | -8 | -4 |
| total stack used | 16 bytes | 8 bytes |
| exit | syscall, RAX=60, RDI=0 | int 0x80, EAX=1, EBX=0 |

## Observations from the real run

- **Grows down, 4 bytes per slot.** `c860 -> c85c -> c858` and back, because in 32-bit mode the
  default operand size for push and pop is 32 bits.
- **Little-endian again:** the low bytes `0x34` and `0x44` sit at the lowest addresses.
- **LIFO holds:** 0x1234 came off first, then 0x11223344.
- **Balanced.** ESP starts and ends at `0xffffc860`.
- **The pops return each register to its original value,** because they pop into the registers the
  values came from. As in 05.asm, LIFO shows up in the memory dump rather than in the final
  registers.
- **No zero-extension surprise.** There is no 64-bit register in the file, so `mov eax,0x11223344`
  (`b8 44 33 22 11`) is the natural encoding. Compare 05.asm, where a bigger constant needed the
  10-byte `movabs`.
- **EBX doubles as the exit status register.** After `xor ebx,ebx` the 0x1234 that was carefully
  round-tripped through the stack is discarded: the register dump goes from ebx=4660 to ebx=0 on
  the last instruction.
- **Addresses vary per machine; the deltas and byte patterns do not.** -4/+4,
  `34 12 00 00` and `44 33 22 11` are fixed by the architecture.


