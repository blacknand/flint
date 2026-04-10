# flint ISA: Instruction Format Specification
**Version: 0.1**

flint uses 32 bits, fixed width registers + instructions. It is little endian. All instructions must be 4-byte aligned. The bottom 2 bits of any valid instruction address are always zero. flint uses nine instruction formats in total. Six are defined here as the base ISA (R, I, S, B, U, J). Three are defined alongside their corresponding GCC backend subsystem work (M, A, V). All formats share the following invariants:
- The opcode field always occupies bits `[31:26]`. The hardware extracts this field identically regardless of the format.
- Where rs1 is present, it always occupies bits `[20:17]`, allowing the hardware to begin reading the register file before format decode is complete.
- In formats where the S-bit is present, it occupies bit `[25]`, immediately to the right of the opcode.
- Bit patterns marked reserved must be zero in all encoded instructions. Behaviour on non-zero reserved fields is undefined.

## Register file
flint has 16 GPRs, `r0` through `r15`, each 32 bits wide. Register fields in all instruction formats are 4-bits wide, encoding values 0-15. `r0` is hardwired to zero. Reading `r0` always produces 0. Writing to `r0` is a silently discarded. This gives rise to several pseudoinstructions described later in this document.

### Registers
| Register | Role | Category |
| :--- | :--- | :--- |
| r0 | Zero | Fixed |
| r1–r4 | Argument / caller-saved | Caller-saved |
| r5–r11 | Callee-saved variables | Callee-saved |
| r12 | Thread pointer | Fixed |
| r13 | Stack pointer | Fixed |
| r14 | Link register | Special |
| r15 | IP scratch | Caller-saved |

## Condition register (CR)
flint maintains a dedicated condition register holding four single-bit flags updated by S-bit instructions:
| Flag | Name | Set when |
| :--- | :--- | :--- |
| N | Negative | Result bit 31 is 1 |
| Z | Zero | Result is zero |
| C | Carry | Unsigned overflow (carry out of bit 31) |
| V | Overflow | Signed overflow (result sign is incorrect) |

CR is only updated by instructions with S=1. Instructions with S=0 leave CR entirely unchanged. CR is never updated by load, store, branch or jump instructions.

## Base formats
| Format | Fields | Primary use |
| :--- | :--- | :--- |
| R | opcode, S, rd, rs1, rs2, funct, reserved | Register-register ALU operations |
| I | opcode, S, rd, rs1, funct, imm | Immediate ALU operations, loads |
| S | opcode, reserved, rs2, rs1, funct, imm | Stores |
| B | opcode, cond, offset | Conditional branches |
| U | opcode, rd, imm | Upper immediate (large constant load) |
| J | opcode, rd, offset | Unconditional jump and link (function calls) |

## R-type (register-register operations)
![R-type format](images/flint%20R-type%20encoding.png)
### Fields
- **opcode** `[31:26]`: identifies this instruction as R-type. One major opcode value covers all R-type operations. The `funct` field completes the decode.
- **S** `[25]`: Set-flags bit. When S=1, CR is updated with the N, Z, C, V flags derived from the result. When S=0, CR is unchanged. Valid on all R-type operations without exception.
- **rd** `[24:21]`: Destination register. The result of the operation is written here.
- **rs1** `[20:17]`: First source register.
- **rs2** `[16:13]`: Second source register. For a NOT operation, this field is unused — the assembler encodes `rs = r0` and the hardware ignores it.
- **funct** `[12:9]`: Selects the specific operation with R-type. 4 bits, 16 slots, 9 currently assigned.
> The potential future use includes shifted-register operations (shift type + shift amount), which would occupy 7 of these 9 bits.
- **Reserved** `[8:0]`: Must be zero. Reserved for future use.

### Operations
Addressing mode 2 is an R-type instruction and the load variant is determined by the funct field.
| funct | Mnemonic | Operation | Notes |
| :--- | :--- | :--- | :--- |
| 0000 | ADD | rd = rs1 + rs2 | |
| 0001 | SUB | rd = rs1 - rs2 | |
| 0010 | AND | rd = rs1 & rs2 | |
| 0011 | OR | rd = rs1 \| rs2 | |
| 0100 | XOR | rd = rs1 ^ rs2 | |
| 0101 | NOT | rd = ~rs1 | rs2 unused, assembler encodes rs2=r0 |
| 0110 | LSL | rd = rs1 << rs2[4:0] | Logical shift left by register |
| 0111 | LSR | rd = rs1 >> rs2[4:0] | Logical shift right, zero-fills |
| 1000 | ASR | rd = rs1 >>> rs2[4:0] | Arithmetic shift right, sign-fills |
| 1001 | — | Reserved | funct=1001 reserved for MUL (M extension) |
| 1010 | — | Unallocated | |
| 1011 | — | LW (load word, register offset) | |
| 1100 | — | LB (load byte signed, register offset) | |
| 1101 | — | LBU (load byte unsigned, register offset) | |
| 1110 | — | LH (load halfword signed, register offset) | |
| 1111 | — | LHU (load halfword unsigned, register offset) | |

### Shift semantics
For LSL, LSR, and ASR, only bits `[4:0]` of the runtime values in `rs2` are used as the shift amount (range 0-31). Bits `[31:5]` of `rs2` are ignored at execution time. This is a semantic constraint.

### Pseudoinstructions
The following assembler mnenomics are accepted and lowered to R-type encodings:
| Pseudo | Expands to | Encoding |
| :--- | :--- | :--- |
| `CMP rs1, rs2` | `SUB r0, rs1, rs2` with S=1 | Result discarded, CR updated |
| `MOV rd, rs1` | `ADD rd, rs1, r0` | Adds zero, effectively copies |
| `NEG rd, rs1` | `SUB rd, r0, rs1` | Zero minus rs1 |

## I-type (immediate operations, loads, JALR)
![I-type format](images/flint%20I-type%20encoding.png)
- **opcode** `[31:26]`: Two distinct major opcode values are used: one for immedaite ALU operations, and one for loads. The `funct` field completes the decode within each group. There is an additional single opcode: JALR.
- **S** `[25]`: Set-flags bit. Valid on immediate ALU operations. Must be zero on load instructions. S=1 on load is invalid encoding. Behaviour is undefined.
- **rd** `[24:21]`: Destination register.
- **rs1** `[20:17]`: Base register. For ALU operations. For loads, the base address register.
- **funct** `[16:14]`: Selects the specific operation within each opcode group. 3 bits, 8 slots per group.
- **imm** `[13:0]`: 14-bit signed immediate. Always sign extend to 32-bits before use. Effective range: -8192 to +8191. For load instructions, imm is the signed byte offset added to `rs1` to form the effective address.

### Immediate ALU operations (opcode Group 1)
| funct | Mnemonic | Operation |
| :--- | :--- | :--- |
| 000 | ADDI | rd = rs1 + sign_extend(imm) |
| 001 | ANDI | rd = rs1 & sign_extend(imm) |
| 010 | ORI | rd = rs1 \| sign_extend(imm) |
| 011 | XORI | rd = rs1 ^ sign_extend(imm) |
| 100 | LSLI | rd = rs1 << imm[4:0] |
| 101 | LSRI | rd = rs1 >> imm[4:0] (logical) |
| 110 | ASRI | rd = rs1 >>> imm[4:0] (arithmetic) |
| 111 | — | Reserved |

> For LSLI, LSRI and ASRI, only bits `[4:0]` of the imm field are used as the shift amount. Bits `[13:5]` of imm must be zero in valid encodings. The assembler enforces this.

### Load operations (opcode Group 2)
Addressing mode 1: effective address = `rs1` + sign_extend(imm).
| funct | Mnemonic | Operation | C type |
| :--- | :--- | :--- | :--- |
| 000 | LB | rd = sign_extend(mem[EA][7:0]) | signed char |
| 001 | LBU | rd = zero_extend(mem[EA][7:0]) | unsigned char |
| 010 | LH | rd = sign_extend(mem[EA][15:0]) | short |
| 011 | LHU | rd = zero_extend(mem[EA][15:0]) | unsigned short |
| 100 | LW | rd = mem[EA][31:0] | int, pointer |
| 101–111 | — | Reserved | |

### Addressing modes 3, 4 and 5
All of these addressing modes are I-type instructions, with their own opcode so they all have the same funct encodings.

| Funct | Instruction |
| :--- | :--- |
| 000 | LB |
| 001 | LBU |
| 010 | LH |
| 011 | LHU |
| 100 | LW |
| 101–111 | Reserved |

- **Mode 3 — pre-increment** (`[rs1 + imm]!`): rs1 is updated before the memory access. Semantics: `rs1 = rs1 + sign_extend(imm)`, then `rd = mem[rs1]`.
- **Mode 4 — post-increment** (`[rs1], imm`): rs1 is updated after the memory access. Semantics: `rd = mem[rs1]`, then `rs1 = rs1 + sign_extend(imm)`.
- **Mode 5 — PC-relative** (`[PC + imm]`): base is implicit PC. rs1 field must be zero (reserved). Semantics: `rd = mem[PC + sign_extend(imm)]`.

### JALR
JALR has a dedicated opcode. JALR is used for unconditional jump and link register. The `funct` field has no purpose here since JALR is the only instruction at this opcode, so it is reserved with the value `000`. JALR gives three pseudoinstructions:
```
RET        →  JALR r0,  r14, 0
CALL reg   →  JALR r14, reg, 0
JR reg     →  JALR r0,  reg, 0
```
#### Semantics
```
target = rs1 + sign_extend(imm)
rd = PC + 4
PC = target
```
The S-bit must be zero (reserved).

> LH and LHU require 2-byte aligned addresses. LW requires 4-byte aligned addresses. LB and LBU have no alignment requirement. Misaligned accesses produce architecturally undefined behaviour.

## S-type (store operations)
![S-type flint encoding](images/S-type%20flint%20encoding.png)
- **opcode** `[31:26]`: One major opcode value for all store operations.
- **Reserved** `[25]`: Must be zero. Produces no result, so the S-bit has no meaning. Bit 25 is reserved rather than repurposed to maintain a consistent visual layout with other formats.
- **rs2** `[24:21]`: Register whose value is written to memory.
- **rs1** `[20:17]`: Base register address. Effective address = `rs1` + sign_extend(imm).
- **funct** `[16:14]`: Selects store width.
- **imm** `[13:0]`: 14-bit signed byte offset. Sign-extended to 32 bits and added to `rs1`. Range: -8192 to +8191. Contiguous field.

| funct | Mnemonic | Operation | Writes |
| :--- | :--- | :--- | :--- |
| 000 | SB | mem[EA] = rs2[7:0] | Low byte of rs2 |
| 001 | SH | mem[EA] = rs2[15:0] | Low halfword of rs2 |
| 010 | SW | mem[EA] = rs2[31:0] | Full word |
| 011–111 | — | Reserved | |

> **Note on signed vs unsigned stores:** There are no SBU or SHU variants. Sign is a property of interpretation on load, not on store. Writing a byte always writes the low 8 bits of `rs2` to memory regardless of signedness — the sign question is resolved when the value is loaded back.

> **Alignment:** SH requires 2-byte aligned EA. SW requires 4-byte aligned EA. SB has no alignment requirement.

### Store addressing mode 3 — pre-increment (`[rs1 + imm]!`)
Uses the same field layout as S-type. Opcode `100100`. rs1 is updated before the memory write.

Semantics: `rs1 = rs1 + sign_extend(imm)`, then `mem[rs1] = rs2`.

| funct | Mnemonic | Writes |
| :--- | :--- | :--- |
| 000 | SB | Low byte of rs2 |
| 001 | SH | Low halfword of rs2 |
| 010 | SW | Full word |
| 011–111 | — | Reserved |

### Store addressing mode 4 — post-increment (`[rs1], imm`)
Uses the same field layout as S-type. Opcode `100101`. rs1 is updated after the memory write.

Semantics: `mem[rs1] = rs2`, then `rs1 = rs1 + sign_extend(imm)`.

| funct | Mnemonic | Writes |
| :--- | :--- | :--- |
| 000 | SB | Low byte of rs2 |
| 001 | SH | Low halfword of rs2 |
| 010 | SW | Full word |
| 011–111 | — | Reserved |

> Store mode 2 (register offset) and store mode 5 (PC-relative) are not defined. Mode 2 would require three register fields which S-type cannot accommodate. Mode 5 is not meaningful for writes.

> **Alignment:** Store modes 3 and 4 follow the same alignment rules as base S-type. The effective address used for the alignment check is the post-writeback address for mode 3 and the pre-writeback address for mode 4.

## B-type (conditional branches)
![B-type encoding](images/flint%20B-type%20encoding.png)

flint uses a CR updated by prior instructions. The branch instruction only needs to know which condition to test — no register fields are required for comparison. This frees the entire remaining word for the offset, yielding a substantially larger branch range than comparison-in-branch designs (such as RV32I (32-bit RISC-V)).
### Fields
- **opcode** `[31:26]`: One major opcode value for all conditonal branches.
- **cond** `[25:22]`: Selects which condition to test against CR. 4 bits, 16 slots, 14 assigned. Is a fixed decode selector baked into the instruction at compile time.
- **offset** `[21:0]`: 22-bit signed PC-relative offset, in units of 4-bytes. The hardware computes the branch target as `PC + sign_extend(offset) << 2`. Effective byte range: ±8MB. The bottom 2 bits of any target address are always zero due to instruction alignment — encoding in units of 4 bytes recovers those bits for range.

### Condition codes
| cond | Mnemonic | Condition tested | Flags |
| :--- | :--- | :--- | :--- |
| 0000 | BEQ | Equal | Z=1 |
| 0001 | BNE | Not equal | Z=0 |
| 0010 | BLT | Signed less than | N != V |
| 0011 | BGE | Signed greater or equal | N=V |
| 0100 | BLO | Unsigned lower | C=0 |
| 0101 | BHS | Unsigned higher or same | C=1 |
| 0110 | BMI | Negative | N=1 |
| 0111 | BPL | Positive or zero | N=0 |
| 1000 | BVS | Overflow set | V=1 |
| 1001 | BVC | Overflow clear | V=0 |
| 1010 | BGT | Signed greater than | Z=0 and N=V |
| 1011 | BLE | Signed less or equal | Z=1 or N != V |
| 1100 | BHI | Unsigned higher | C=1 and Z=0 |
| 1101 | BLS | Unsigned lower or same | C=0 or Z=1 |
| 1110–1111 | — | Reserved | |

## U-type (upper immediate)
![U-type encoding](images/U-type%20flint%20encoding.png)

U-type loads a large immediate into the upper portion of a register, enabling the construction of arbitrary 32-bit constants via a two-instruction sequence with a subsequent ADDI.
### Fields
- **opcode** `[31:26]`: One major opcode value. A second opcode value is reserved for an AUIPC variant (add upper immediate to PC), needed for positon-independent code.
- **rd** `[25:22]`: Destination register.
- **imm** `[21:0]`: 22-bit unsigned immediate. Placed into bits `[31:14]` of `rd`. Bits `[13:0]` of `rd` are cleared to zero. No S-bit.
#### 32-bit constant construction
To load an aribtrary 32-bit constant K into `rd`:
```
LUI     rd, upper(K)        ; rd = upper18(K) << 14
ADDI    rd, rd, lower(K)    ; rd = rd + sign_extend(lower14(K))
```
> **Sign adjustment:** Because ADDI sign-extends its 14-bit immediate, if bit 13 of `lower14(K)` is 1, ADDI interprets the lower part as a negative number and subtracts. The assembler must detect this and add 1 to the upper immediate to compensate. This adjustment is handled transparently by the assembler — the programmer specifies the constant and the assembler emits the correct pair. Formally: if `K[13] == 1`, encode `upper = (K >> 14) + 1`, else `upper = K >> 14`.

## J-type (jump and link)
![J-type encoding](images/J-type%20encoding.png)

### Fields
- **opcode** `[31:26]`: One major opcode value.
- **rd** `[25:22]`: Return address destination. The hardware writes `PC + 4` (the address of the instruction following the JAL) into `rd` before jumping. Writing to `r0` discards the return address silently.
- **offset** `[21:0]`: 22-bit signed PC-relative offset, in units of 4-bytes. Branch target = `PC + sign_extend(offset) << 2`. Range ±8MB.
### Semantics
```
rd = PC + 4
PC = PC + sign_extend(offset) << 2
```
Both operations occur atomically — `rd` receives the return address before the jump takes effect.
#### Usage patterns
**Function call:** `rd` is the designated link register `r14`:
```
JAL r14, target     ; r14 = return address, PC = target
```
> The assembler may expose these as separate mnemonics (`CALL target` and `JMP target`) as pseudoinstructions over the single JAL encoding.

**Unconditional jump:** `rd` is `r0`, return address discarded:
```
JAL r0, target      ; PC = target, return address silently discarded
```

## Opcode space allocation
6-bit opcode field, 64 total slots.
> I have not assigned the opcodes their concrete binary values yet. This will be done when I begin working on the assembler.

| Opcode range | Assigned to |
| :--- | :--- |
| 000000 | R-type (all register-register ops) |
| 000001 | I-type immediate ALU |
| 000010 | I-type loads (addressing mode 1) |
| 000011 | S-type stores (addressing mode 1) |
| 000100 | B-type branches |
| 000101 | U-type upper immediate |
| 000110 | U-type AUIPC (reserved, Phase 3 PIC) |
| 000111 | J-type jump and link |
| 001000 | JALR dedicated opcode |
| 001001–001111 | Reserved for M-type (addressing modes, Phase 3) |
| 010000–010111 | Reserved for A-type (atomics, Phase 3) |
| 011000–011111 | Reserved for V-type (vectors, Phase 3) |
| 100000 | Reserved for M extension (MUL/DIV) |
| 100001 | Load addressing mode 3 (`[rs1 + imm]!`) |
| 100010 | Load addressing mode 4 (`[rs1], imm`) |
| 100011 | Load addressing mode 5 (`[PC + imm]`) |
| 100100 | Store addressing mode 3 (`[rs1 + imm]!`) |
| 100101 | Store addressing mode 4 (`[rs1], imm`) |
| 100110–111111 | Unallocated, available for future extensions |