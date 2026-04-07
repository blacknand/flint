# flint Calling Convention
> See the flint ISA specification first.
## Registers
| Register | Role | Category |
| :--- | :--- | :--- |
| r0 | Zero (hardwired) | Fixed |
| r1–r4 | Arguments / return value / caller-saved scratch | Caller-saved |
| r5–r11 | Callee-saved variables / frame pointer when needed | Callee-saved |
| r12 | Thread pointer (TLS base) | Fixed |
| r13 | Stack pointer | Fixed |
| r14 | Link register (return address) | Special |
| r15 | Intra-procedure-call scratch (veneer use) | Caller-saved |

## Argument passing
- First four registers go in r1-r4 in order
- Arguments beyond four are pushed onto the stacj in the caller's outgoing argument space, in order, starting at `[sp+0]` on the callee's entry
- Return value goes in r1. 64-bit return values use r1 (low) and r2 (high)

## The stack
- Grows downward: push by subtracting from sp, pop by adding
> 8-byte alignment required before any call
- sp is always 4-byte aligned
- sp after prologue points to the bottom of the current frame and never moves until the epilogue

## Frame layout
From high address to low address:
```
┌──────────────────────────┐ ← previous sp (frame top)
│  saved lr                │  4 bytes (omitted in leaf functions)
│  saved r5–r11            │  4 bytes each, only registers actually used
│  spilled locals          │  compiler-assigned offsets from sp
│  outgoing argument space │  max stack args across all calls in function
└──────────────────────────┘ ← new sp (frame bottom, set once in prologue)
```

### Prologue responsibilites
1. Subtract frame size from sp (one instruction, computed statically at compile time)
2. Save lr if function is non-leaf
3. Save any callee-saved registres the function uses
4. Optionally set r5 as frame pointer when frame size is dynamic (i.e., `alloca()`)

### Epilogue responsibilites
1. Restore all callee-saved registers
2. Restore lr if it was saved
3. Add frame size back to sp
4. Execute RET (psuedoinstruction)