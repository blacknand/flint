# flint
**A custom 32-bit RISC ISA and complete GCC backend — built to understand compiler infrastructure from first principles.**

flint is a from-scratch instruction set architecture and the toolchain infrastructure around it: a GCC backend, an assembler, and a cycle-level pipeline simulator. Most GCC backends are ported, but I decided to completely build the flint compiler from scratch. Every instruction, every encoding choice, every ABI decision was made deliberately with a specific compiler engineering question in mind: "what does GCC need to see here to exercise the relevant subsystem correctly?". The goal of this is to understand the GCC backend as in depth as possible.

The result is a 32-bit RISC processor with features that span the full breadth of the GCC backend: 16 general-purpose registers and 8 128-bit SIMD registers exercising multi-bank register allocation; five addressing modes including pre- and post-increment giving the addressing mode selection logic something real to do; predicated execution and if-conversion; a counted hardware loop; load-exclusive/store-exclusive atomics with a proper weak memory model; and full position-independent code and thread-local storage support.

## Documentation
See `/docs` for the flint documentation. 

## Acknowledgments
- The flint ISA, specification, etc. is influenced by RISC-V (RV32I) and AArch32. This includes the RISC-V unpriviledged specification, RISC-V calling, and AArch32 GPRs, PC and special-purpose registers.
- I used Claude to learn concepts + topics quickly. I also used claude to assist in some of the design decisions, such as the encoding for the types of operations in the flint ISA. Claude did not write any of the code, it was used more as a learning tool/search engine.