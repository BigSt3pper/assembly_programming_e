# 03.asm - call, ret and the Return Address (64-bit)

## Summary

`_start` puts 10 in RAX and calls `calculate`, which adds 5 and returns. The point is not the sum
but the return address: `call` pushes the address of the next instruction, and `ret` pops it into
RIP. This README shows that 8-byte value read straight out of memory.

## Build and run

```bash
nasm -f elf64 -g -F dwarf 03.asm -o 03.o
ld 03.o -o 03
./03 ; echo "EXIT=$?"
```

Real output: no output, `EXIT=0`.

Trace, including the extra breakpoint at `calculate` that the assignment asks for:

```bash
gdb -q -batch -ex 'set $bits = 64' -ex 'set $dump_words = 2' \
    -ex 'set $dump_bytes = 0' -ex 'set $extra_break = "calculate"' \
    -x trace_engine.py ./03
```

RSP at `_start` = `0x7fffffffd6f0`.

## Where the labels are

This is the punch line, so check it before reading the trace:

```
_start      0x401000      mov eax,0xa
            0x401005      call calculate
after_call  0x40100a      <- the return address will be this
calculate   0x401015      add rax,5
            0x401019      ret
```

`nm 03` agrees: `40100a t after_call` and `401015 t calculate`.

## Step-by-step trace

| # | Instruction | RSP before | RSP after | Delta | What happened |
|---|---|---|---|---|---|
| 1 | `mov eax,0xa` | `d6f0` | `d6f0` | 0 | RAX = 10. No stack use. |
| 2 | `call calculate` | `d6f0` | `d6e8` | -8 | Pushes return address `0x40100a`, jumps to calculate. |
| 3 | `add rax,0x5` | `d6e8` | `d6e8` | 0 | RAX = 15. Stack untouched. |
| 4 | `ret` | `d6e8` | `d6f0` | +8 | Pops `0x40100a` into RIP. |
| 5 | `nop` | `d6f0` | `d6f0` | 0 | Resumed at after_call. RAX = 15 as the comment says. |
| 6 | `mov eax,0x3c` | `d6f0` | `d6f0` | 0 | RAX = 60. |
| 7 | `xor rdi,rdi` | `d6f0` | `d6f0` | 0 | RDI = 0. |
| - | `syscall` | `d6f0` | - | - | Not stepped: it would kill the traced process. |

Registers after `ret`: RAX=15, RBX=0, RCX=0, RDX=0.

## Stack memory at the nop (after ret has run)

RSP = `0x7fffffffd6f0`, RBP = `0x0`.

```
0x7fffffffd6f0  0x0000000000000001   <- RSP   kernel's argc
0x7fffffffd6f8  0x00007fffffffda03           kernel's argv[0]
```

The return address is no longer in the live stack: `ret` popped it, moving RSP past it.

## The return address, dumped inside calculate

A second run stops on entry to `calculate`, before `add rax,5` executes. This is the `x/2gx $rsp`
dump the assignment asks for:

```
RSP  = 0x7fffffffd6e8
0x7fffffffd6e8:  0x000000000040100a   0x0000000000000001
top of stack (the return address) = 0x000000000040100a
it points at:    0x40100a <after_call>:  nop
return address - _start = 10 bytes into the program
```

So inside `calculate` the stack holds exactly one thing of ours:

```
0x7fffffffd6e8  0x000000000040100a   <- RSP   return address -> after_call
0x7fffffffd6f0  0x0000000000000001           kernel's argc
```

There is no saved RBP and no local here: `calculate` has no prologue. The `call` is the only thing
in the whole program that touches the stack.

## Observations from the real run

- **The pushed value is a code address, and it is the instruction after the call.** `0x40100a` is
  `_start+10`, because the `call` itself is 5 bytes at `0x401005`.
- **This is why call and ret need no bookkeeping.** `ret` does not know where to go; it pops
  whatever 8 bytes are on top into RIP. It only works because call and ret are balanced. Leave one
  extra value on the stack inside a function and `ret` jumps to that instead: the classic crash.
- **One -8, one +8, so the stack is balanced** at `0x7fffffffd6f0`.
- **Data travelled in RAX, not on the stack.** `calculate` received 10 in RAX and left 15 in RAX.
  The stack carried control flow; registers carried data.
- **RBP never changed** (stayed `0x0`), confirming no frame is needed.
- **Surprise: `mov rax, 10` became `mov eax,0xa`.** The usual NASM narrowing, harmless because a
  32-bit write zero-extends into RAX.
- **Surprise: the call is relative, but the stack gets an absolute address.** The encoding is
  `e8 0b 00 00 00`, a +0x0b displacement from the next instruction (`0x40100a + 0x0b = 0x401015`).
  The instruction is position-independent; the pushed value is a real absolute address.
- **Addresses vary per machine; the offset does not.** The return address is always `_start+10`
  here, and the RSP deltas are always -8/+8.

