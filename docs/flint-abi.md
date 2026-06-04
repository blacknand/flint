# flint Application Binary Interface
**Version: 0.2**

See flint-isa.md for the instruction set specification. This document specifies the calling convention, stack layout, and ELF identification for flint v0.1.

## Registers
| Register | Role | Category |
| :--- | :--- | :--- |
| r0 | Zero (hardwired) | Fixed |
| r1–r4 | Arguments / return value / caller-saved scratch | Caller-saved |
| r5–r11 | Callee-saved variables / frame pointer when needed | Callee-saved |
| r12 | Caller-saved scratch | Caller-saved |
| r13 | Stack pointer | Fixed |
| r14 | Link register (return address) | Special |
| r15 | Caller-saved scratch | Caller-saved |

## Argument passing

### Scalar arguments
- The first four argument words go in r1–r4 in order.
- Arguments beyond what fits in r1–r4 are pushed onto the stack in the caller's outgoing argument space, in order, starting at `[sp+0]` on the callee's entry.

### 64-bit arguments
flint uses **register-pair alignment**: a 64-bit value must occupy an aligned pair of argument registers, specifically r1:r2 or r3:r4. A 64-bit value cannot span r2:r3.

The allocation algorithm walks through the argument list and, for each argument:
1. If the argument is 64-bit and the next available register is r2 or r4, skip that register (it remains unused and is not available to subsequent arguments).
2. Allocate from the next available aligned pair.
3. If no aligned pair fits in r1–r4 (e.g., a 64-bit argument arriving after r1, r2, r3 are all used), the argument goes on the stack — flint does not split a 64-bit value across a register and the stack.

The low word of a 64-bit pair goes in the lower-numbered register (low → r1 or r3, high → r2 or r4). flint is little-endian and this matches.

### Aggregate (struct/union) arguments
- Aggregates of size ≤ 8 bytes are passed by value in registers, following the same allocation rules as scalar 64-bit values (aligned pair if 8 bytes; single register if ≤ 4 bytes).
- Aggregates larger than 8 bytes are passed **by hidden pointer**: the caller allocates space, the pointer to it occupies one argument slot, and the callee dereferences as needed.

## Return values
- Scalar return value in r1.
- 64-bit return value in r1 (low) and r2 (high).
- Aggregate returns larger than 8 bytes: the caller allocates the return buffer and passes a hidden pointer to it as an implicit first argument in r1. The function returns void; the result is materialized through the hidden pointer.

## Variadic arguments
- Use the same register/stack allocation rules as regular arguments, including the 64-bit pair-alignment rule.
- On entry to a variadic function, the callee saves all unused argument registers (those after the last named argument) to a save area immediately above the on-stack argument storage. This produces a single contiguous block of argument storage that `va_list` can walk.
- `va_list` is implemented as a pointer that advances through this contiguous block as arguments are consumed.

## The stack
- Grows downward — push by subtracting from sp, pop by adding.
- sp is always 4-byte aligned.
- 8-byte aligned at the point of any call instruction.
- sp is set once at the end of the prologue and does not move until the epilogue.
- **No red zone.** flint does not define a region below sp that is safe from signal/interrupt handlers. Leaf functions must adjust sp normally before storing below it.

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

## Prologue responsibilities
The prologue is emitted in the following order:

1. **Lower sp by the frame size.** For frame sizes ≤ 8192 bytes, use a single `ADDI sp, sp, -frame_size`. For larger frames, materialize the offset via `LUI` + `ADDI` into a scratch register (r12 or r15 is typical) and then `SUB sp, sp, scratch`.
2. **Save lr** if the function is non-leaf: `SW r14, [sp + lr_offset]`.
3. **Save callee-saved registers** the function uses, in ascending register order (r5 first, then r6, ..., up to r11): one `SW` per register at its assigned offset.
4. **Set the frame pointer** if the frame size is dynamic (the function uses `alloca` or has a variable-length array). flint conventionally uses r5 as the frame pointer; when so used, r5 is loaded with the value of sp at the end of the prologue and is not available as a general-purpose callee-saved register within that function.

## Epilogue responsibilities
The epilogue reverses the prologue:

1. **Restore callee-saved registers** in reverse order (r11 first, then r10, ..., down to r5).
2. **Restore lr** if it was saved.
3. **Raise sp by the frame size**: single `ADDI sp, sp, frame_size` for small frames, or the materialize-and-add sequence for large frames.
4. **Return**: `JALR r0, r14, 0` (assembled as `RET`).

## Calling convention summary

| Property | Value |
| :--- | :--- |
| Argument registers | r1–r4 |
| Return value register(s) | r1 (or r1:r2 for 64-bit) |
| Caller-saved (volatile) | r1–r4, r12, r15 |
| Callee-saved (non-volatile) | r5–r11 |
| Stack pointer | r13 |
| Link register | r14 |
| Stack growth direction | Downward |
| Stack alignment at call | 8 bytes |
| 64-bit pair alignment | r1:r2 or r3:r4 only |
| Large-aggregate convention | Hidden pointer |
| Red zone | None |

## ELF identification
- `e_machine`: `0x9F1A` — flint architecture identifier, chosen from the unallocated range above `0x9000`.

## Out of scope for v0.1
The following are explicit non-features of v0.1. See flint-extensions.md.

- **Thread-local storage**: no TLS base register is reserved in v0.1. (r12 was previously sketched as a TLS register in early drafts of flint; it is now general caller-saved scratch.)
- **Position-independent code**: no AUIPC, no GOT/PLT conventions. v0.1 supports static linking only.
- **Stack unwinding metadata**: DWARF CFI emission is left to a later phase. The frame layout above is regular enough that simple unwinders can walk it without CFI by following saved-lr offsets, but standard DWARF support is the eventual goal.