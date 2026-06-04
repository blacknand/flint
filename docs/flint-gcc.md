# flint GCC Backend Structure

This document describes the GCC backend directory layout, the files within it, and the role of each. For the phased implementation plan (what to write first, what to defer), see flint-phases.md.

## Directory layout
```
gcc/config/flint/
├── flint.h           # Target macros (register classes, ABI parameters, type sizes)
├── flint.cc          # Target hook implementations (C++)
├── flint.md          # Machine description: RTL patterns and assembly templates
├── flint.opt         # Command-line option definitions
├── constraints.md    # Register-class and immediate-value constraints
├── predicates.md     # RTL predicate definitions over operand shapes
└── t-flint           # Makefile fragment
```

Outside this directory, one shared file is also edited:
- `gcc/config.gcc`: registers `flint*-*-elf*` as a recognized target triple and points the build at the directory above.

## File responsibilities

### flint.h
A collection of `#define`d target macros GCC reads at compile time to learn the shape of the target. Macro families needed for the minimum backend:

- **Type and pointer sizes**: `BITS_PER_WORD = 32`, `UNITS_PER_WORD = 4`, `POINTER_SIZE = 32`, `INT_TYPE_SIZE = 32`, `LONG_TYPE_SIZE = 32`, `LONG_LONG_TYPE_SIZE = 64`, and so on.
- **Register file definition**: `FIRST_PSEUDO_REGISTER = 16`, the `FIXED_REGISTERS` and `CALL_USED_REGISTERS` initialiser arrays, `REG_ALLOC_ORDER`, the `REGISTER_NAMES` table mapping register numbers to assembly names.
- **Register classes**: the `enum reg_class` declaration via `REG_CLASS_CONTENTS`, `REG_CLASS_NAMES`, `N_REG_CLASSES`. For the minimum backend, two classes suffice: `GENERAL_REGS` (the allocatable registers) and `ALL_REGS` (everything).
- **Stack and ABI parameters**: `STACK_POINTER_REGNUM = 13`, `FRAME_POINTER_REGNUM` (the soft frame pointer, eliminated), `STACK_BOUNDARY = 32`, `PREFERRED_STACK_BOUNDARY = 64` (8-byte at calls), `STACK_GROWS_DOWNWARD = 1`, `FRAME_GROWS_DOWNWARD = 1`.
- **Eliminable registers**: `ELIMINABLE_REGS` declares the soft FP elimination to either the hard FP (when one is needed) or sp.
- **Calling convention parameters**: `STACK_POINTER_OFFSET`, `FIRST_PARM_OFFSET`, function-argument macros that delegate to `flint.cc` via hooks.
- **Assembler dialect knobs**: `ASM_OUTPUT_LABELREF`, `ASM_COMMENT_START`, `REGISTER_PREFIX` (likely `"r"` or empty), `LOCAL_LABEL_PREFIX`.

### flint.cc
Implementations of target hooks declared in the generic GCC machinery. Hooks needed for the minimum backend:

- `TARGET_FUNCTION_ARG`, `TARGET_FUNCTION_ARG_ADVANCE`, `TARGET_FUNCTION_VALUE`: walk the calling convention. These implement the rules in flint-abi.md — argument register allocation, 64-bit pair alignment, return value placement.
- `TARGET_RETURN_IN_MEMORY`: returns true for aggregate returns > 8 bytes (hidden-pointer convention).
- `TARGET_LEGITIMATE_ADDRESS_P`: recognizes valid memory operand shapes. For v0.1, just `(reg)` and `(plus reg const_int)` where the constant fits in 14 bits signed.
- `TARGET_PRINT_OPERAND`, `TARGET_PRINT_OPERAND_ADDRESS`: control assembly output for `%`-substitutions in MD pattern templates.
- `flint_expand_prologue`, `flint_expand_epilogue`: emit the prologue/epilogue RTL as described in flint-abi.md. Called from the `prologue` and `epilogue` named patterns in `flint.md`.
- Constant materialization helper: a function that, given a 32-bit integer and a destination register, emits the optimal instruction sequence — single ADDI when the value fits, single LUI when the low 14 bits are zero, LUI+ADDI pair otherwise (with sign adjustment).
- `TARGET_RTX_COSTS`: a rudimentary cost model so the combine and simplification passes make sensible choices.

### flint.md
The machine description. Written in GCC's MD language, a Lisp-like DSL describing RTL templates and the assembly they emit. Each pattern declares its RTL shape, its operand constraints, and the assembly output template.

Patterns needed for the minimum backend (added incrementally across phases A–D):

- **Moves**: `movsi`, `movhi`, `movqi`.
- **Integer arithmetic**: `addsi3`, `subsi3`.
- **Logical**: `andsi3`, `iorsi3`, `xorsi3`, `one_cmplsi3`.
- **Shifts**: `ashlsi3`, `lshrsi3`, `ashrsi3` (with both register and 5-bit-immediate variants).
- **Extensions**: `extendqisi2`, `extendhisi2`, `zero_extendqisi2`, `zero_extendhisi2`. These are synthesized via shift pairs (sign-extend) or AND-with-mask (zero-extend) since flint v0.1 has no dedicated extension instructions.
- **Memory**: load and store patterns parameterized over modes via `<mode>` iterators.
- **Control flow**: `cbranchsi4` (compare-and-branch), `jump` (unconditional), conditional-branch helper patterns for each condition code.
- **Calls**: `call`, `call_value`, indirect-call variants.
- **Function entry/exit**: `prologue`, `epilogue`, `return` (or `simple_return`).
- **Constant splitter**: turns a large `(set reg const_int)` into a LUI+ADDI sequence.

### flint.opt
Command-line option definitions in GCC's `.opt` mini-language. For v0.1 this file is essentially empty — no `-m` flags beyond what comes from the generic option machinery. (Later phases might add `-mno-mul` or similar to opt out of any extensions that get added.)

### constraints.md
Register-class and immediate-value constraints referenced by MD patterns. Each constraint is a one-letter (or `:`-prefixed multi-letter) tag with a description and a Boolean predicate.

Constraints needed for v0.1:
- `r`: any general-purpose register (member of `GENERAL_REGS`).
- `I`: 14-bit signed immediate (fits in ADDI).
- `K`: 5-bit unsigned shift amount (fits LSLI/LSRI/ASRI).
- `J`: the zero constant.
- `M`: 18-bit immediate suitable for LUI (low 14 bits zero).

### predicates.md
RTL predicate functions used in MD pattern operands. Each predicate is a C function returning true if its RTX argument is a valid operand of that kind.

Predicates needed for v0.1:
- `arith_operand`: a register or a 14-bit signed immediate.
- `branch_operand`: a register (since flint branches compare register-register only).
- `call_operand`: a symbol reference or a register.
- `move_operand`: any valid source for a move (register, memory, constant).

### t-flint
A small Makefile fragment listing any backend-specific source files beyond `flint.cc` (none, in the minimum backend) and any multilib configurations (none in v0.1). A typical t-flint is three or four lines.

## Build target
Configure GCC for `flint-elf` (or `flint-unknown-elf`) and use newlib for the C library. Static linking only in v0.1; no shared libraries, no dynamic loader.

A separate binutils port (assembler, linker, `objdump`, `readelf`) is also needed to assemble and link the GCC output. That work is outside the scope of these documents.