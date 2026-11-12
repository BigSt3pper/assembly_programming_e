# 04.asm - A Stack Frame in 32-bit Mode (BITS 32, int 0x80)

## Summary

The 32-bit twin of 02.asm: `_start` calls `work`, which saves EBP, reserves an 8-byte local area,
stores 100 and 200, reads them back, restores ESP and returns. Same logic as 02.asm, but every
stack slot is 4 bytes and the program exits with `int 0x80`.

## Build and run

```bash
nasm -f elf32 -g -F dwarf 04.asm -o 04.o
ld -m elf_i386 04.o -o 04
./04 ; echo "EXIT=$?"
```

Real output: no output, `EXIT=0`.

`-m elf_i386` is required: the default `ld` targets x86-64 and rejects a 32-bit object file.
`file 04` confirms a 32-bit executable.

Trace:

```bash
gdb -q -batch -ex 'set $bits = 32' -ex 'set $dump_words = 4' \
    -ex 'set $dump_bytes = 0' -ex 'set $extra_break = 0' \
    -x trace_engine.py ./04
```

ESP at `_start` = `0xffffc860`.

## Step-by-step trace

| # | Instruction | ESP before | ESP after | Delta | What happened |
|---|---|---|---|---|---|
| 1 | `call work` | `c860` | `c85c` | -4 | Pushes return address `0x08049005` (4 bytes). |
| 2 | `push ebp` | `c85c` | `c858` | -4 | Saves caller's EBP (0x0). |
| 3 | `mov ebp,esp` | `c858` | `c858` | 0 | EBP = frame base. |
| 4 | `sub esp,0x8` | `c858` | `c850` | -8 | Reserves two 4-byte local slots. |
| 5 | `mov [ebp-0x4],0x64` | `c850` | `c850` | 0 | Local 1 = 100 stored at `c854`. |
| 6 | `mov [ebp-0x8],0xc8` | `c850` | `c850` | 0 | Local 2 = 200 stored at `c850` (= ESP). |
| 7 | `mov eax,[ebp-0x4]` | `c850` | `c850` | 0 | EAX = 100. |
| 8 | `mov ebx,[ebp-0x8]` | `c850` | `c850` | 0 | EBX = 200. |
| 9 | `nop` | `c850` | `c850` | 0 | Marker for the memory dump below. |
| 10 | `mov esp,ebp` | `c850` | `c858` | +8 | Discards both locals. |
| 11 | `pop ebp` | `c858` | `c85c` | +4 | Restores EBP = 0x0. |
| 12 | `ret` | `c85c` | `c860` | +4 | Pops `0x08049005` into EIP. |
| 13 | `mov eax,0x1` | `c860` | `c860` | 0 | EAX = 1 = exit syscall number (32-bit). |
| 14 | `xor ebx,ebx` | `c860` | `c860` | 0 | EBX = 0 = exit status (32-bit). |
| - | `int 0x80` | `c860` | - | - | Not stepped: it would kill the traced process. |

All addresses share the prefix `0xffff`. Registers at the end: EAX=1, EBX=0, EBP=0x0.

## Stack memory at the nop, inside work

ESP = `0xffffc850`, EBP = `0xffffc858`.

```
0xffffc850  0x000000c8   <- ESP   local [ebp-8] = 200
0xffffc854  0x00000064           local [ebp-4] = 100
0xffffc858  0x00000000   <- EBP   saved EBP (was 0)
0xffffc85c  0x08049005           return address into _start
0xffffc860  0x00000001           kernel's argc = 1
0xffffc864  0xffffca03           kernel's argv[0]
```

Same shape as 02.asm, half the scale. Locals below EBP, saved EBP at `[ebp]`, return address at
`[ebp+4]` (not `+8`).

## 04.asm vs 02.asm

| Item | 02.asm (64-bit) | 04.asm (32-bit) |
|---|---|---|
| stack slot | 8 bytes | 4 bytes |
| call cost | -8 | -4 |
| push ebp/rbp cost | -8 | -4 |
| local area | 16 bytes | 8 bytes |
| return address | `0x401005` | `0x08049005` |
| code addresses near | `0x4010xx` | `0x08049xxx` |
| exit | syscall, EAX-style: RAX=60, RDI=0 | int 0x80: EAX=1, EBX=0 |

## Observations from the real run

- **Step size is 4, and that is the main difference.** Every call, push, pop and ret moved ESP by
  exactly 4, because in 32-bit mode the stack pointer and default operand size are 32 bits.
- **Still grows down:** `c860 -> c85c -> c858 -> c850`, then back up in the epilogue.
- **Balanced.** ESP starts and ends at `0xffffc860`.
- **EBP returns to 0x0.** As in 02.asm, `push ebp` saved zero because `_start` has no caller frame;
  it is restored faithfully anyway.
- **No zero-extension surprise.** This file has no 64-bit register, so `mov eax,100` really is the
  native instruction (`b8 64 00 00 00`). The "NASM shortened mov rax to mov eax" surprise from
  01-03 cannot happen in 32-bit code.
- **The int 0x80 ABI differs from syscall.** Exit is EAX=1 with the status in EBX, not RAX=60 with
  the status in RDI. Using the 64-bit convention here would have exited with whatever was in EBX.
- **Code sits near 0x08049000 and the stack near 0xffffc860.** Both fit in 32 bits, which is why
  the return address occupies exactly one 4-byte slot.
- **Addresses vary per machine; the deltas and layout do not.** -4/+4 and the frame layout are
  fixed by the code and the 32-bit mode.

