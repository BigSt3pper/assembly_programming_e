# 01.asm - Push and Pop (LIFO) on the 64-bit Stack

## Summary

Puts 100 in RAX and 200 in RBX, pushes both, then pops them into RCX and RDX. It shows the
stack as a last-in-first-out store, and proves the order by which value lands in which register.

## Build and run

```bash
nasm -f elf64 -g -F dwarf 01.asm -o 01.o
ld 01.o -o 01
./01 ; echo "EXIT=$?"
```

Real output: no output, `EXIT=0`.

Trace:

```bash
gdb -q -batch -ex 'set $bits = 64' -ex 'set $dump_words = 4' \
    -ex 'set $dump_bytes = 0' -ex 'set $extra_break = 0' \
    -x trace_engine.py ./01
```

RSP at `_start` = `0x7fffffffd6f0`.

## Step-by-step trace

| # | Instruction | RSP before | RSP after | Delta | What happened |
|---|---|---|---|---|---|
| 1 | `mov eax,0x64` | `d6f0` | `d6f0` | 0 | RAX = 100. No stack use. |
| 2 | `mov ebx,0xc8` | `d6f0` | `d6f0` | 0 | RBX = 200. No stack use. |
| 3 | `push rax` | `d6f0` | `d6e8` | -8 | RSP lowered by 8, then 0x64 written there. |
| 4 | `push rbx` | `d6e8` | `d6e0` | -8 | RSP lowered by 8, then 0xc8 written there. |
| 5 | `nop` | `d6e0` | `d6e0` | 0 | Marker for the memory dump below. |
| 6 | `pop rcx` | `d6e0` | `d6e8` | +8 | 200 read into RCX, RSP raised by 8. |
| 7 | `pop rdx` | `d6e8` | `d6f0` | +8 | 100 read into RDX, RSP raised by 8. |
| 8 | `nop` | `d6f0` | `d6f0` | 0 | RSP back at its start: balanced. |
| 9 | `mov eax,0x3c` | `d6f0` | `d6f0` | 0 | RAX = 60 = exit syscall number. |
| 10 | `xor rdi,rdi` | `d6f0` | `d6f0` | 0 | RDI = 0 = exit status. |
| - | `syscall` | `d6f0` | - | - | Not stepped: it would kill the traced process. |

All addresses share the prefix `0x7fffffff`. Registers at the end: RAX=60, RBX=200, RCX=200, RDX=100.

## Stack memory at the first nop

RSP = `0x7fffffffd6e0`, RBP = `0x0`.

```
0x7fffffffd6e0  0x00000000000000c8   <- RSP  200, pushed last  (rbx)
0x7fffffffd6e8  0x0000000000000064           100, pushed first (rax)
0x7fffffffd6f0  0x0000000000000001           kernel's argc = 1
0x7fffffffd6f8  0x00007fffffffda03           kernel's argv[0]
```

The lowest address holds the newest value, because the stack grows down. Everything from
`d6f0` upward is the kernel's start-up stack, already there before `_start` ran.

After the pops, RSP is back at `d6f0` and the dump shows the kernel's argc/argv again. The bytes
`0xc8` and `0x64` are still physically at `d6e0` and `d6e8`: `pop` moves RSP only, it does not
erase memory. They are now above RSP, so the next push overwrites them.

## Observations from the real run

- **Grows down, 8 bytes per slot.** Measured deltas are exactly -8 and +8, because in 64-bit mode
  push and pop work on 8-byte operands.
- **LIFO is proven by the registers.** 200 was pushed second and came off first into RCX; 100 came
  off second into RDX. Swap the two `pop` lines and the values swap with them.
- **Balanced.** RSP starts and ends at `0x7fffffffd6f0`, net change 0: two pushes (-16) cancelled
  by two pops (+16). A function that fails to do this corrupts the stack for `ret`.
- **RBP is never used.** It stays `0x0` for the whole program, so this program has no frame and no
  frame pointer.
- **Surprise: NASM did not emit what was written.** The source says `mov rax, 100` but the bytes
  are `b8 64 00 00 00`, which is `mov eax,0x64`. NASM uses the 5-byte 32-bit form because the
  value fits, and a 32-bit write zero-extends into the full RAX, so the result is still
  `0x0000000000000064`. Same for `mov rax, 60` -> `mov eax,0x3c`.
- **But `xor rdi, rdi` was not shortened** (`48 31 ff`, REX.W kept). Narrowing a move-immediate to
  32 bits is safe; narrowing a register operation would destroy the upper 32 bits.
- **Push is two steps: lower RSP, then store at the new RSP.** Pop is the mirror: read, then raise
  RSP. That ordering is why the newest value sits at the lowest address.
- **Addresses are machine-specific.** `0x7fffffffd6f0` is this machine's start-up stack address
  (pinned by `set disable-randomization on` so the trace repeats). Another machine gives a
  different value in the same neighbourhood. The deltas and the layout do not change.

