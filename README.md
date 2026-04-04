# flint
**A custom 32-bit RISC ISA and complete GCC backend — built to understand compiler infrastructure from first principles.**

Flint is a from-scratch instruction set architecture and the toolchain infrastructure around it: a GCC backend, an assembler, and a cycle-level pipeline simulator. Most GCC backends are ported, but I decided to completely build the flint compiler from scratch. Every instruction, every encoding choice, every ABI decision was made deliberately with a specific compiler engineering question in mind: "what does GCC need to see here to exercise the relevant subsystem correctly?". The goal of this is to understand the GCC backend as in depth as possible.

The result is a 32-bit RISC processor with features that span the full breadth of the GCC backend: 15 general-purpose registers and 8 128-bit SIMD registers exercising multi-bank register allocation; five addressing modes including pre- and post-increment giving the addressing mode selection logic something real to do; predicated execution and if-conversion; a counted hardware loop; load-exclusive/store-exclusive atomics with a proper weak memory model; and full position-independent code and thread-local storage support.

## Documentation
See `/docs` for the flint documentation. There you will find the ISA specification document.