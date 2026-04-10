# flint Application Binary Interface
> See flint-isa.md for the instruction set specification.
 
## Registers
| Register | Role | Category |
| :--- | :--- | :--- |
| r0 | Zero (hardwired) | Fixed |
| r1–r4 | Arguments / return value / caller-saved scratch | Caller-saved |
| r5–r11 | Callee-saved variables / frame pointer when needed | Callee-saved |
| r12 | Thread pointer (TLS base) | Fixed |
| r13 | Stack pointer | Fixed |
| r14 | Link register (return address) | Special |
| r15 | IP scratch (veneer use) | Caller-saved |
 
## Condition register (CR) classification
CR is modelled by GCC as hard register 16 (after r0–r15). It has the following ABI properties:
 
- **Fixed** — GCC never allocates CR directly. No instruction names it as a source or destination register field.
- **Call-clobbered** — there is no requirement to preserve CR across a call. A callee may execute S=1 instructions freely. The caller cannot assume CR survives a call.
- **Never explicitly saved or restored** — GCC tracks CR liveness through the condition code machinery. If a CR value must survive a call, GCC materialises it into a GPR before the call.
 
> **TODO (Phase 3):** Document LC (loop counter) caller/callee classification once the hardware loop extension is designed.
 
> **TODO (Phase 3):** Document VR (vector register) calling convention once the V extension is designed.
 
## Argument passing
- First four arguments go in r1–r4 in order
- Arguments beyond four are pushed onto the stack in the caller's outgoing argument space, in order, starting at `[sp+0]` on the callee's entry
- Return value goes in r1. 64-bit return values use r1 (low) and r2 (high)
 
> **TODO (before Phase 2):** Struct passing rules — when does a struct go in registers, when on the stack, when split?
 
> **TODO (before Phase 2):** Variadic argument passing rules — required before implementing `va_list`.
 
## The stack
- Grows downward: push by subtracting from sp, pop by adding
- sp is always 4-byte aligned
- 8-byte alignment required at the point of any call instruction
- sp after prologue points to the bottom of the current frame and never moves until the epilogue
- **No red zone.** flint does not define a red zone below sp. There is no region below the stack pointer that is guaranteed safe from signal handlers or interrupt handlers. Leaf functions must adjust sp normally.
 
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
 
### Prologue responsibilities
1. Subtract frame size from sp (one instruction, computed statically at compile time)
2. Save lr if function is non-leaf
3. Save any callee-saved registers the function uses
4. Optionally set r5 as frame pointer when frame size is dynamic (i.e., `alloca()`)
 
### Epilogue responsibilities
1. Restore all callee-saved registers
2. Restore lr if it was saved
3. Add frame size back to sp
4. Execute RET (pseudoinstruction)
 
## ELF identification
- `e_machine`: `0x9F1A` — flint architecture identifier, chosen from the unallocated range above `0x9000`.