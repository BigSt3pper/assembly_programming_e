# 02.asm - A Full Stack Frame: call, Prologue, Locals, Epilogue (64-bit)

## Summary

`_start` calls `work`, which saves RBP, sets up a frame pointer, reserves 16 bytes of locals,
stores 100 and 200 in them, reads them back, then tears the frame down and returns. It shows the
whole life cycle of a stack during a function call.

## Build and run

```bash
nasm -f elf64 -g -F dwarf 02.asm -o 02.o
ld 02.o -o 02
./02 ; echo "EXIT=$?"
```

Real output: no output, `EXIT=0`.

Trace:

```bash
gdb -q -batch -ex 'set $bits = 64' -ex 'set $dump_words = 4' \
    -ex 'set $dump_bytes = 0' -ex 'set $extra_break = 0' \
    -x trace_engine.py ./02
```

RSP at `_start` = `0x7fffffffd6f0`. Addresses below drop the `0x7fffffff` prefix after the first.

## Step-by-step trace

| # | Instruction | RSP before | RSP after | Delta | What happened |
|---|---|---|---|---|---|
| 1 | `call work` | `d6f0` | `d6e8` | -8 | Pushes return address `0x401005`, jumps to work. |
| 2 | `push rbp` | `d6e8` | `d6e0` | -8 | Saves caller's RBP (0x0). |
| 3 | `mov rbp,rsp` | `d6e0` | `d6e0` | 0 | RBP = frame base, fixed for the whole function. |
| 4 | `sub rsp,0x10` | `d6e0` | `d6d0` | -16 | Reserves two 8-byte local slots. No memory written. |
| 5 | `mov [rbp-0x8],0x64` | `d6d0` | `d6d0` | 0 | Local 1 = 100 stored at `d6d8`. Store, not push. |
| 6 | `mov [rbp-0x10],0xc8` | `d6d0` | `d6d0` | 0 | Local 2 = 200 stored at `d6d0` (= RSP). |
| 7 | `mov rax,[rbp-0x8]` | `d6d0` | `d6d0` | 0 | RAX = 100 read back. |
| 8 | `mov rbx,[rbp-0x10]` | `d6d0` | `d6d0` | 0 | RBX = 200 read back. |
| 9 | `nop` | `d6d0` | `d6d0` | 0 | Marker for the memory dump below. |
| 10 | `mov rsp,rbp` | `d6d0` | `d6e0` | +16 | Epilogue: discards both locals in one move. |
| 11 | `pop rbp` | `d6e0` | `d6e8` | +8 | Restores RBP = 0x0. |
| 12 | `ret` | `d6e8` | `d6f0` | +8 | Pops `0x401005` into RIP, back in `_start`. |
| 13 | `mov eax,0x3c` | `d6f0` | `d6f0` | 0 | RAX = 60. |
| 14 | `xor rdi,rdi` | `d6f0` | `d6f0` | 0 | RDI = 0. |
| - | `syscall` | `d6f0` | - | - | Not stepped: it would kill the traced process. |

Registers at the end: RAX=60, RBX=200, RBP=0x0.

## Stack memory at the nop, inside work

RSP = `0x7fffffffd6d0`, RBP = `0x7fffffffd6e0`.

```
0x7fffffffd6d0  0x00000000000000c8   <- RSP   local [rbp-16] = 200
0x7fffffffd6d8  0x0000000000000064           local [rbp-8]  = 100
0x7fffffffd6e0  0x0000000000000000   <- RBP   saved RBP (was 0)
0x7fffffffd6e8  0x0000000000401005           return address into _start
0x7fffffffd6f0  0x0000000000000001           kernel's argc = 1
0x7fffffffd6f8  0x00007fffffffda03           kernel's argv[0]
```

Three layers, from the bottom of the listing up:

1. Locals `[rbp-8]` and `[rbp-16]`, below RBP, inside the 16 bytes that `sub rsp,16` reserved.
2. Saved RBP, sitting exactly at `[rbp]`.
3. The return address at `[rbp+8]`, above the saved RBP, plus the kernel's start-up data.

Note there is no slack: `[rbp-16]` is exactly RSP, because 16 bytes were reserved and 16 used.

## Observations from the real run

- **call is push plus jump.** It cost -8, the same as a push, because the CPU pushed the return
  address `0x401005`. The value on top of the stack right after a call is a code address.
- **ret is pop plus jump.** +8, and the popped `0x401005` goes into RIP. That is the entire return
  mechanism: no hidden return-value slot, just an address parked on the stack by `call`.
- **sub rsp,16 moved RSP by 16 in one instruction,** not two pushes. Reserving locals in bulk is
  cheap. Memory stays untouched until the two stores at steps 5 and 6.
- **Locals are addressed off RBP, not RSP,** which is the point of a frame pointer: RSP may move
  during the body, RBP does not, so `[rbp-8]` keeps meaning the same slot.
- **The epilogue is size-proof.** `mov rsp,rbp` throws away whatever was reserved without naming a
  number, so changing `sub rsp,16` to `sub rsp,64` needs no edit in the epilogue.
- **Balanced.** RSP starts and ends at `0x7fffffffd6f0`. The -8/-8/-16 setup is undone by
  +16/+8/+8.
- **RAX and RBX survive the call** as 100 and 200, proving the round trip through the locals worked.
- **Saved RBP was 0x0, and that is correct.** `_start` is entered by the kernel, not called, so
  there is no caller frame. Saving and restoring zero is still the right thing to do.
- **Addresses are machine-specific; the layout is not.** Locals below RBP, saved RBP at `[rbp]`,
  return address at `[rbp+8]` - that is fixed by the code, not by the machine.

