> Note: extremly minimal and the bare minimum ABI of what is needed for a minimal GCC backend.

# flint ABI v0.3

Calling convention and stack specification for flint v0.3. See flint-isa.md for the instruction set.

---

## Registers

| Register | Role | Saver |
|:---|:---|:---|
| r0 | Zero (hardwired) | — |
| r1–r4 | Arguments / return values | Caller-saved |
| r5–r11 | General purpose variables | Callee-saved |
| r12 | Scratch | Caller-saved |
| r13 | Stack pointer | Fixed |
| r14 | Link register | Special |
| r15 | Scratch | Caller-saved |

**Caller-saved** (volatile): r1–r4, r12, r15. The callee may freely overwrite these. If the caller needs their values after a call, it must save them before the call.

**Callee-saved** (non-volatile): r5–r11. If a function uses any of these it must save them on entry and restore them before returning.

**Fixed**: r0 (hardwired zero), r13 (stack pointer). These are never available for general allocation.

**Special**: r14 (link register). JAL and JALR write the return address here. A non-leaf function must save r14 to the stack before making any call, and restore it before returning.

---

## Argument passing

The first four argument words go in r1–r4 in order. Arguments beyond four words are pushed onto the stack in the caller's outgoing argument space, starting at [sp+0] on the callee's entry, in order.

Return value is in r1. A 64-bit return value uses r1 (low word) and r2 (high word).

---

## Stack

- Grows downward — push by subtracting from sp, pop by adding.
- sp is always 4-byte aligned.
- sp must be 8-byte aligned at the point of any call instruction.
- sp is set once at the end of the prologue and does not move until the epilogue.
- No red zone.

---

## Frame layout

```
┌──────────────────────┐  ← previous sp (frame top)
│  saved r14 (lr)      │  4 bytes, only if non-leaf function
│  saved r5–r11        │  4 bytes each, only registers actually used
│  spilled locals      │  compiler-assigned offsets
│  outgoing args       │  space for stack arguments to callees
└──────────────────────┘  ← new sp (set once in prologue)
```

---

## Out of scope for v0.3

- 64-bit argument passing and register-pair alignment rules
- Struct/aggregate argument and return conventions
- Variadic argument handling and va_list layout
- Position-independent code conventions
- Thread-local storage
- DWARF CFI / stack unwinding metadata
- ELF identification (e_machine value)