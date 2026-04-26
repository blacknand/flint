# flint
**A custom 32-bit RISC ISA and complete GCC backend, built to understand the GCC compiler infrastructure from first principles.**

flint is a from-scratch instruction set architecture and the toolchain infrastructure around it: a GCC backend, an assembler, and a cycle-level pipeline simulator. I built flint to understand the GCC backend from first principles, since there is no better way to gain an understanding of a large scale system like GCC than learning how to navigate and modify it. There is also no textbooks on the inner workings of GCC, and GCC is very difficult to learn and contribute to as a beginner, so hopefully others looking to learn and contribute to GCC may also find this useful.

## Patches
See the patches made to GCC and binutils-gdb:
- [flint-GCC](https://github.com/blacknand/gcc/tree/flint)
- [flint-binutils-gdb](https://github.com/blacknand/binutils-gdb/tree/flint)

## Documentation
See `/docs` for the flint documentation. 

## Contributing
flint is more of an educational project than anything else, so it is very likely to have flaws or issues. If you find any, please either raise an issue or open up a PR. Thank you.

## Acknowledgments
- The flint ISA, specification, etc. is influenced by RISC-V (RV32I) and AArch32. This includes the RISC-V unpriviledged specification, RISC-V calling, and AArch32 GPRs, PC and special-purpose registers.
- I used Claude to learn concepts + topics quickly. I also used claude to assist in some of the design decisions, such as the encoding for the types of operations in the flint ISA. Claude did not write any of the code, it was used more as a learning tool/search engine.
