# flint Deferred Features

This document tracks features explicitly out of scope for v0.1. Each is listed with a brief rationale for its deferral and a sketch of what its eventual implementation would entail. The intent is to keep the v0.1 ISA and backend lean while preserving room for incremental learning later.

The opcode allocation in flint-isa.md leaves slots `001000` through `111111` unallocated. New instructions draw from this pool. Bit `[25]` of R-type and I-type instructions is reserved across v0.1 so that flag-setting variants can be added without breaking existing encodings.

---

## Instruction set extensions

### M extension (multiply / divide)
Hardware integer multiply and divide.

**Rationale for deferral**: not required for correctness. Without an `M` extension, GCC emits libcalls into libgcc (`__mulsi3`, `__divsi3`, `__udivsi3`, `__modsi3`, `__umodsi3`) for these operations. Every multiplication or division becomes a function call — slow, but correct.

**Implementation sketch**: a single new major opcode value for MUL/DIV operations, encoded as R-type (three register fields). MD patterns `mulsi3`, `mulhisi3` (high half of unsigned multiply), `divsi3`, `udivsi3`, `modsi3`, `umodsi3`. The funct slot 1001 in R-type was provisionally reserved for MUL in earlier drafts; assigning a dedicated opcode is cleaner.

### A extension (atomics)
Load-reserved / store-conditional or atomic read-modify-write primitives.

**Rationale for deferral**: flint v0.1 is single-core only. Atomic operations are required only for multi-threaded code and require a defined memory model first.

**Implementation sketch**: opcodes for LR, SC, and possibly atomic-add/and/or/xor. MD patterns for the `__atomic_*` builtin family. Memory model specification (sequential consistency vs weaker orderings).

### V extension (vector / SIMD)
Vector registers and operations.

**Rationale for deferral**: substantial extension touching the register file, the calling convention, and many GCC backend subsystems (vectorization passes, vector cost models, etc.). Out of scope until the scalar backend is solid.

**Implementation sketch**: a separate vector register file (typically 16–32 registers), vector load/store instructions, element-wise arithmetic, reduction operations. New MD modes (`V4SI`, `V8HI`, etc.), vectorizer cost hooks, and the `TARGET_VECTORIZE_*` hook family.

---

## Architectural features

### Condition flags and S-bit
flint v0.1 reserves bit `[25]` of R-type and I-type instructions for a future S-bit. Re-introducing flag-setting ALU variants is feasible without re-encoding the base ISA.

**Rationale for deferral**: significant backend cost — CCmode handling, paired patterns for each ALU operation, careful interaction with combine for compare-into-branch fusion. The simpler compare-and-branch design in v0.1 is the right starting point for learning MD patterns; flags can be added once the basics are solid.

**Implementation sketch**: define a 4-bit condition register (N, Z, C, V) as a hard register adjacent to the GPR file. Add MD patterns for flag-setting variants of each ALU operation (paired with the existing non-flag-setting variants). Implement `SELECT_CC_MODE`, possibly multiple `CCmode`s (`CC_NZmode`, `CC_NZCVmode`, `CC_Cmode`) selected by which flags a downstream branch consumes. Define the comparison-from-flags branch patterns. Update combine to recognize compare-then-branch sequences and fuse them.

### Conditional select (CSEL)
A conditional select instruction: `rd = cond ? rs1 : rs2`.

**Rationale for deferral**: not required for correctness. Without CSEL, GCC's if-conversion pass cannot turn short branches into branchless sequences, but the compiled code is still correct. CSEL requires either condition flags or in-instruction comparison fields — neither is present in v0.1.

**Implementation sketch**: most naturally added alongside the condition-flags extension. Encode as R-type with the condition selector in funct. Define the `movsicc` MD pattern to enable GCC's if-conversion.

---

## Addressing modes

flint v0.1 has a single memory addressing mode: `[rs1 + imm14]`. Several useful additional modes were sketched in earlier drafts.

### Register-offset (mode 2)
Loads with effective address = `rs1 + rs2`. Useful for indexed array access.

**Rationale for deferral**: a separate MD pattern family. Encodes naturally in R-type (three register fields, a width selector in funct) but stores would need a non-S-type encoding since S-type has only two register fields.

**Implementation sketch**: extend the R-type funct space to encode loads (funct = 1011 through 1111 for LW, LB, LBU, LH, LHU at register offset). A new major opcode for register-offset stores, in an R-type-shaped encoding with rs2 as the value, rs1 as the base, and a third register field as the offset. Extend `flint_legitimate_address_p` to accept `(plus reg reg)`.

### Pre-increment (mode 3) and post-increment (mode 4)
`[rs1 + imm]!` and `[rs1], imm`. ARM-style auto-modify addressing.

**Rationale for deferral**: substantial backend complexity. GCC represents these via `(pre_modify)` / `(post_modify)` RTX wrapped around the MEM operand. Recognizing them in `TARGET_LEGITIMATE_ADDRESS_P` is straightforward; getting LRA (the register allocator) to correctly handle the same register being both base and writeback target requires care. The auto-inc-dec pass needs to be enabled, and the cost model needs to be tuned to make it find opportunities.

**Implementation sketch**: separate opcodes for pre-modify and post-modify loads/stores. MD patterns using `(pre_modify ...)` and `(post_modify ...)` around the memory operand. Set `HAVE_PRE_MODIFY_DISP` and `HAVE_POST_MODIFY_DISP` macros.

### PC-relative load (mode 5)
`[PC + imm]`. Enables literal pools placed near a function.

**Rationale for deferral**: the 14-bit immediate gives ±8 KB range from PC, which means assembler literal-pool management — splitting long functions so that all PC-relative references stay in range. Closer to assembler work than to interesting GCC backend work.

**Implementation sketch**: a separate opcode, encoded as I-type with rs1 forced to zero. MD patterns expressed via `UNSPEC` since GCC doesn't have a native PC-relative-load RTX. Assembler responsible for pool emission and range tracking.

---

## Compilation features

### Position-independent code (PIC)
AUIPC (add upper immediate to PC), GOT-based symbol references, PLT-based external calls.

**Rationale for deferral**: substantial backend work spanning multiple subsystems. flint v0.1 targets static linking only — no shared libraries, no dynamic loader.

**Implementation sketch**: the U-type reserved bits `[3:0]` accommodate an AUIPC opcode-variant selector. Add an `AUIPC rd, imm` instruction with semantics `rd = PC + (imm << 14)`. MD patterns for `@GOT` and `@GOTPLT` symbol references using AUIPC + LW sequences. ABI extensions describing the GOT layout and PLT stub conventions.

### Thread-local storage (TLS)
`__thread`-qualified variables.

**Rationale for deferral**: requires a designated thread-pointer register and TLS-relocation sequences. Not useful in single-threaded contexts and not required for any C program that doesn't explicitly use `__thread`.

**Implementation sketch**: designate a thread-pointer register (r12 is a candidate, currently used as scratch). Emit TLS-relocation sequences in MD patterns for `(unspec UNSPEC_TLS ...)` references. Add `TARGET_HAVE_TLS` macro. Linker and dynamic-loader support required for fully-dynamic TLS models.

---

## Convenience instructions

### Sign and zero extension
SXTB, UXTB, SXTH, UXTH.

**Rationale for deferral**: not required for correctness. Without dedicated instructions, GCC emits `LSLI rd, rs, 24; ASRI rd, rd, 24` for `extendqisi2` (sign-extend byte) and similar two-instruction sequences for the other extensions. Slow but correct.

**Implementation sketch**: a handful of R-type funct slots assigned to extension operations. Replace the existing two-instruction synthesized patterns with single-instruction patterns. Trivial.

### Bit-manipulation instructions
CLZ (count leading zeros), CTZ (count trailing zeros), POPCNT.

**Rationale for deferral**: not required for correctness. GCC handles `__builtin_clz` etc. via libcalls when no native pattern is defined.

**Implementation sketch**: R-type funct assignments, `clzsi2`, `ctzsi2`, `popcountsi2` MD patterns.

---

## Process

When a deferred feature is implemented, the change set should update:
- **flint-isa.md**: the instruction encoding, the opcode allocation table, any affected register-file or invariant text.
- **flint-abi.md**: if the calling convention changes (TLS register, PIC conventions, vector argument passing).
- **flint-extensions.md** (this file): move the implemented item to a "released" section or remove it.
- **flint-phases.md**: add a new phase E, F, ... describing the implementation work and the testable milestone.

Keeping these documents consistent as the project evolves is itself good practice for working on a real toolchain — GCC's own backend documentation has the same maintenance burden.