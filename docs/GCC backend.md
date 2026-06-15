# GCC backend
> I wrote this while building flint-GCC. My main references were the RV32I backend (32-bit RISC-V), or1k and GCC-VAM. I used Claude + Gemini to piece it all together and work out what everything meant so it is highly likely to have errors. I also wrote this before realising I could just use the GCC INT manual for referencing the or1k backend!

In this document I cover all of the basics needed to build a minimal functional GCC backend to enable users to compile C programs on their target machine. This document is specific to flint, a toy RISC 32 bit architecture I desired to learn GCC backend development -- that being said, you should easily be able to read it and apply it specifically to your target machine (that is, if you are trying to build one). It is written assuming absoultely **no prior knowledge** of GCC but does assume *some* familiartiy with systems programming, C/C++, etc. 

# Target hooks
A GCC hook is an object-oriented interace implemented in C++. It is the mechanism by which the generic, machine-independent core of the compiler (the middle-end) interrogates the specific, machine-dependent backend about its physical hardware capabilities.

Historically, GCC relied almost entirely on C preprocessor macros to configure the backend. If the middle-end needed to know if the hardware supported a specific feature, it evaluated an `#ifdef`. This creates several issues such as no type checking, static evaluation -- macros are evaluated at compile time of the compiler itself which made it incredibly difficult to build a single GCC binary that could dynamically switch between different hardware capabilities at runtime -- and code bloat. To solve this target hooks were introduced.

Instead of querying a macro, the generic compiler code calls a C function via a global structure, `targetm` -- for target machine. Here is an overview of how the system is built and executed:

#### 1. The definition (`target.def`)
All hooks are defined in a central file, `target.def`. `.def` files are written in a domain specific language (DSL). The generated `targetm` C structure defines the hooks name, its return type, its arguments, and a fallback default function.

#### 2. The default implementations (`targhooks.cc`)
If a backend does not implement a specific target hook, GCC instead uses a default function. `targetm` points to a default function in `targhooks.c`.

#### 3. The backend override (`<your-isa>.cc`)
In your backends primary C++ file, you define the functions specific to your ISA. At the bottom of the file, you instantiate the hooks you want to override and bind them to the `targetm` structure.

# `<target>.h`
In this section I will go over the each component in the `<target>.h` file required for a minimal GCC backend.

#### `TARGET_CPU_CPP_BUILTINS()`
```cpp
/* Names to predefine in the preprocessor for this target machine.  */
#define TARGET_CPU_CPP_BUILTINS()		\
  do						\
    {						\
      builtin_define ("__flint__");		\
      builtin_define ("__FLINT__");	\
      builtin_assert ("cpu=flint");		\
      builtin_assert ("machine=flint");		\
    }						\
  while (0)
```

When GCC compiles a C file, it runs a preprocessor over it first. The preprocesor has a set of predefined macros, such as: `__linux__`, `__arm__`, `__riscv` which let the C code do things like:


```cpp
#ifdef __flint__
        // flint-specific path
#endif
```

`TARGET_CPU_CPP_BUILTINS` is the hook GCC calls during preprocessor initialistion to inject your targets predefined macros. Its called once per compilation unit before any user code is processed. 

`builtin_define` injects a `#define` and `builtin_assert` injects a predicate assertion.

#### Storage layout
The storage layout block tells GCC the fundemental memory and word-size properties of the target machine. Every size decision GCC makes for data types propogates from this section.

The following block of code is self explanatory, it just defines the basic storage properties of the target machine:

```cpp
#define DEFAULT_SIGNED_CHAR 1
#define BITS_BIG_ENDIAN 0
#define BYTES_BIG_ENDIAN 0
#define WORDS_BIG_ENDIAN 0
#define BITS_PER_WORD 32
#define UNITS_PER_WORD 4
#define POINTER_SIZE 32
```

There are two types of distinct alignment in flint:
- **Instruction alignment:** a constraint on the program counter. Every flint instruction is exactly 4 bytes wide, and the ISA requires to sit at 4-byte aligned addresses. This is a property of instruction fetch. The hardware's instruction fetch unit assumes it can always read a full 32-bit instruction word from a 4-byte aligned address -- if the PC were ever odd, the fetch unit would be reading a partial instruction which is nonsensical. This is enforced in GCC by `FUNCTION_BOUNDARY 32` which enforces functions starting on a 32-bit boundary. It is also enforced by the assembler that will refuse to assemble instructions at misaligned offsets.
- **Data alignment:** a constraint on memory operands. When you execute `LW r1, r13, 0` the effective address is `r13 + 0` which must be 4-byte aligned. When you execute `LH r1, r13, 2` the effective address must be 2-byte aligned. This is what `STRICT_ALIGNMENT 1` is for.

The reason `STRICT_ALIGNMENT` matters is because GCC sometimes wants to do things like copy a struct byte by byte or access a field at an unaligned offset. On x86 the hardware would allow this byt on flint it produces undefined behaviour. `STRICT_ALIGNMENT 1` tells GCC it must always ensure natural alignment -- it will do something like insert padding instead of emitting a misaligned access.

The other non-obvious storage layout macros are worth diving into:
- `BIGGEST_ALIGNMENT 32`: the largest alignment GCC will ever give any object, in bits.
- `FUNCTION_BOUNDARY 32`: the alignment of function entry points, in bits. Functions start at 4-byte aligned addresses so that the first function fetch is valid.
- `PARM_BOUNDARY 32`: the minimum alignment of function arguments passed on the stack, in bits. Each stack argument occupies at least a 4-byte aligned slot. This means even a `char` argument pushed onto the stack gets padded to 4 bytes, because the load on the callee side will use `LW`.
- `STACK_BOUNDARY 32`: the minimum alignment of the stack pointer itself, in bits. 
> It can be confusing as to what the `STACK_BOUNDARY` and `PREFERRED_STACK_BOUNDARY` macros are for. The flint ABI states that `sp` must be 8-byte aligned at call sites yet `STACK_BOUNDARY` is 4-bytes. This is why: `STACK_BOUNDARY` tells GCC the minimum alignment the stack pointer maintains throughout the function. The 8-byte at call sites rule is a stronger, point-in-time constraint that only applies at the moment a call instruction executes. Enforcing 8-byte alignment always (setting `STACK_BOUNDARY 64`) would waste space -- every function would over-align its frame even if it never touches a 64-bit value. Enforcing it only at call sites is the pragmatic middle ground: the caller adjusts sp to an 8-byte boundary before executing the call instruction, and the callee gets a cleanly-aligned stack on entry.
- `PREFERRED_STACK_BOUNDARY`: preferred alignment. If a `long long` argument is passed into a function call, then the 64-bit value requires 8-byte alignment. If the callee recieves a `long long` argument on the stack, or if it spills a `long long` to the stack, the compiler needs to know the stack was 8-byte aligned on entry so it can place the value correctly. 

##### How GCC uses these to make decisions
```cpp
struct s {
        char a;
        int b;
};
```

GCC needs to lay this out in memory. a is 1 byte. b is 4 bytes and requires 4-byte alignment. If GCC placed b immediately after an at offset 1, it would be at a misaligned address. Because of `STRICT_ALIGNMENT 1`, GCC knows it cannot do this — an LW to offset 1 of the struct would be undefined on flint. So GCC inserts 3 bytes of padding between a and b, making b sit at offset 4. The struct is 8 bytes total.


On x86 with STRICT_ALIGNMENT 0, the same layout decision is made by convention for performance, but GCC could in principle emit the misaligned access and the hardware would handle it.


##### Relative cost of operations
> Think about what a `LB` instruction has to do. The memory system fetches a naturally aligned 4-byte chunk from memory. It then has to extract the single byte the programmer asked for from said 4-byte chunk, and sign-extend it to fill the full 32-bit register. That extraction and sign-extension costs something like extra cycles. Compare that to `LW` which fetches 4 bytes, and puts them directly in a register.
`SLOW_BYTE_ACCESS` determines the decision GCC makes when it wants to either zero-initialise or copy memory. It has to choose to either do it byte by byte or word by word. When set to 1, GCC prefers to use word-width operations whenever possible, even if that menans doing slightly more work to align things first.

GCC will still emit `LB`/`SB` when the C code explicitly operates on a `char`, but it won't choose byte-at-a-time sequences as its preferred approach for things like struct copies or memory zeroing. For example, GCC may generate different code for something like `memset(buf, 0, 16)`: with 0, GCC might emit 16 `SB` instructions to zero 16 bytes -- one store per byte. With 1, GCC prefers to emit 4 `SW` instructions -- one-4-byte store per word, covering all 16 bytes in 4 operations rather than 16. It ends with the same result but with four times fewer instructions.

#### `REG_ALLOC_ORDER`
`REG_ALLOC_ORDER` determines the order in which GCC allocates registers. flint allocates the caller-saved registers first and then callee-saved. The reason for this is because callee-saved registers come with a cost: if GCC allocates r5 for use inside a function, that function must emit code to save r5 on entry and restore it on exit -- that's the ABI contract. Even if r5 is only used for one small computation, you must pay the save/restore cost unconditionally.

Caller-saved registers carry no such obligation. If GCC allocates r12 inside a function, the function can just use it freely. The ABI says the caller is responsible for preserving anythign it cares about across a call, so the callee ows nothing.

By allocating caller-saved registers first, GCC minimises the number of save/restore pairs it has to emit in prologues and epilogues.It only reaches into the callee-saved set (r5-r11) when it has run out of caller-saved options, and when it does, it knows it must emit the save/restore for each one it uses.

### Register classes
A register class is a named set of registers that share some capabilitiy. When GCC has an RTL instruction that requires a register with a particular capability -- like a base address register, it uses the class system to constrain which phsycial registers are valid for that operand. The instruction says "this operand must come from class X" and GCCs register allocator only assigns registers that are members of X.

### GAS Assembly 
> This is specific to GAS directives but general enough for assembly as a language (without being machine specific).
The three fundamental sections are:
- `.text` — executable code lives here. Read-only at runtime on most OSes.
- `.data` — initialised globals live here (int x = 5). The actual value 5 is stored in the object file and loaded into memory.
- `.bss` — uninitialised globals live here (int y). The key insight: the object file doesn't store any bytes for these — it just records "reserve N bytes of zeroed memory here." The OS or startup code zeros it at load time. This makes object files smaller.

When GCC emits assembly it switches between sections using directives — lines that start with . and are instructions to the assembler, not machine instructions. For example:

> The macros `TEXT_SECTION_ASM_OP` that exist in `<target>.h` are defined in `varsam.cc`, some of them are required by GCC but others are handled by default.


```asm
.section .text
foo:
        add r1, r2, r3
        ret

        .section .data
x:
        .word 5

        .section .bss
y:
        .space 4
```

The other (main) sections are `.rodata` for read-only data (const) and `.sbss` for small uninitialised globals. GCC by default reroutes read-only data to `data_section` instead, so it is not explicitly needed in a toy backend -- unless you care about `const`s being in read-only non-code memory. 

#### `ASM_OUTPUT_ALIGN()`
> When the assembler processes a .s file, it maintains an internal counter tracking the current address being written to. Every instruction or data byte emitted advances it. This means the address needs to be aligned.


The `.balign N` directive means align to N bytes, unambiguously. The `b` stands for "bytes". It is the safest and portable option as oppose to `.align N` which is machine dependent.

### Calling convention (ABI)
GCC processes arguments one at a time, left to right. As it processes each one it needs to decide: does this argument go in a register, or on the stack? For flint, the first four arguments go in r1, r2, r3, r4. Any arguments beyond four go on the stack. 

When GCC is about to place an argument, N, it looks at the counter and asks: is N <= 4? if yes, put in the next argument register. If no, put it on the stack.

This is what `CUMULATIVE_ARGS` and `INIT_CUMULATIVE_ARGS` are for. 

### Trampolines
C supports nested functions as a GCC extension. A nested function is a function defined inside another function:
```c
int outer(int x) {
        int inner(int y) {
                return x + y;
        }
        return inner(5);
}
```
`inner` can access `x` even though `x` lives in `outer`'s stack frame. This works fine when `outer` calls `inner` directly since GCC can arrange for `inner` to find `x` via a static chain pointer stored in a dedicated register (SC_REGNUM in flint). The problem arises when you take the address of a nested function and pass it somewhere else like a callback:
```c
void call_it(int (*fn)(int));

int outer(int x) {
        int inner(int y) { return x + y; }
        call_it(inner);
}
```

`call_it` recieves a plain function pointer. When ti calls through that pointer, it has no idea it needs to set up a static chain register first. It just jumps to the address and calls it like a normal function. But `inner` needs the static chain pointer to find `x` -- without it, `x + y` reads garbage. 

This is what a trampoline is used for. Instead of passing the raw address of `inner`, GCC generates a small piece of executable code -- the trampoline -- which:
1. Loads the correct static chain value into SC_REGNUM (r15)
2. Jumps to `inner`

The function pointer passed to `call_it` points to the trampoline, not to `inner` directly. When `call_it` calls through the pointer, it hits the trampoline, which sets up r15 and then jumpts to `inner`. `inner` finds its static chain correcly. The trampoline is generated at runtime, on the stack, because it contains a specific address (`inner`) and a specific value (the static chain pointer for this particular call to `outer`) that aren't known until runtime. 

### `ELIMINABLE_REGS` and the stack pointer
When GCC is compiling a function, it needs to generate memory addresses for two kinds of things:
- **Local variables**: things like `int x = 5;` that live in the current frame
- **Incoming arguments:** the values the caller passed in, which live just above the current frame.

Both of these are addressed relative to the stack pointer. The issue is that the stack pointer on function entry and after the prologue are different:
```
high addresses
┌─────────────────┐
│  incoming args  │
├─────────────────┤  ← sp on function entry
│  saved registers│
│  local variables│
│  outgoing args  │
└─────────────────┘  ← sp after prologue
```

After the prologue decrements the stack pinter, the stack pointer points to the bottom of the frame. To access a local variable at some offset, GCC would need to know the exact total frame size to compute the right offset from SP, but GCC doesn't know the total frame size while it's in the middle of compiling the function -- it hasn't finished allocating all the locals yet. GCC needs to emit addresses for locals and arguments before it knows the final frame size.

GCC solves this with a virtual reference point. Instead of comitting to a real SP-relative address immediately, GCC generates all frame references relative to two virtual registers:
- **The virtual frame pointer (SFP)**: used as a stable reference point for locals and saved registers. Conceptually sits at the top of the current frame.
- **The virtual argument pointer (AP)**: used to address incoming arguments. Conceptually sits at the boundary between the caller's frame and the current frame.

During compilation, every access to a local variable is expressed as `SFP + offset` and every access to an incoming argument is expressed as `AP + offset`. GCC doesn't need to know the real frame size yet -- it just emits these virtual-register-relative addresses throughout the function body. 

After the frame layout is fully determined and register allocation has occured, GCC runs the register elimination pass, replacing every reference to SFP and AP with a real register + an corrected offset. That replacement is what `flint_initial_elimination_offset()` computes.

#### `ACCUMULATE_OUTGOING_ARGS`
When function A calls function B and needs to pass arguments on the stack, those arguments need to go somewhere. There are two strategies:
- **Push/pop model**: A pushes each stack argument immediately before the call and pops them after. sp moves up and down around every call site. This is the classic x86 approach.
- **Accumulate model**: GCC analyses the entire function body, finds the maximum stack space needed for outgoing arguments across all call sites in the function, and allocates that space once in the prologue. sp never moves mid-function — the outgoing argument area is always there at the bottom of the frame, and A just writes arguments into it before each call.

flint uses the accumulate model, which is why the frame layout looks like this:
```
high addresses
┌─────────────────┐  ← sp on entry / AP / SFP
│  saved registers│
│  local variables│
│  outgoing args  │  ← pre-allocated, fixed size
└─────────────────┘  ← sp after prologue, stays here
```
The benefit is that sp is completely stable after the prologue — which simplifies elimination and makes the frame layout predictable. The cost is that you always pay for the maximum outgoing argument space even if most call sites need less.
