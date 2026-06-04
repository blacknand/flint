# flint ISA v0.3

32-bit fixed-width instructions, little-endian, 16 registers. All instructions are 4-byte aligned — the bottom 2 bits of any valid instruction address are always zero.

The opcode field always occupies bits `[31:26]`. Where rs1 is present it always occupies bits `[20:17]`. Reserved bits must be zero; behaviour on non-zero reserved fields is undefined.

No condition register. No flags. Comparisons are folded into branch instructions.

---

## Register file

16 GPRs, r0–r15, each 32 bits wide. Register fields are 4 bits wide. r0 is hardwired zero — reads always return 0, writes are silently discarded.

| Register | Role | Saver |
|:---|:---|:---|
| r0 | Zero (hardwired) | — |
| r1–r4 | Arguments / return values | Caller-saved |
| r5–r11 | General purpose variables | Callee-saved |
| r12 | Scratch | Caller-saved |
| r13 | Stack pointer | Fixed |
| r14 | Link register | Special |
| r15 | Scratch | Caller-saved |

---

## Instruction formats summary

| Format | Use |
|:---|:---|
| R | Register-register ALU |
| I | Immediate ALU, loads |
| S | Stores |
| B | Conditional branches |
| U | Upper immediate (large constants) |
| J | Unconditional jump and link |
| JALR | Jump and link register (indirect) |

---

## R-type

```
31      26 25    22 21    18 17    14 13    10 9         0
[ opcode ] [rd      ] [rs1    ] [rs2    ] [funct  ] [reserved ]
    6           4          4          4        4           10
```

One opcode value covers all R-type operations. The `funct` field selects the specific operation.

| funct | Mnemonic | Operation |
|:---|:---|:---|
| 0000 | ADD | rd = rs1 + rs2 |
| 0001 | SUB | rd = rs1 − rs2 |
| 0010 | AND | rd = rs1 & rs2 |
| 0011 | OR  | rd = rs1 \| rs2 |
| 0100 | XOR | rd = rs1 ^ rs2 |
| 0101 | NOT | rd = ~rs1 (rs2 must be r0) |
| 0110 | LSL | rd = rs1 << rs2[4:0] |
| 0111 | LSR | rd = rs1 >> rs2[4:0] (logical, zero-fills) |
| 1000 | ASR | rd = rs1 >>> rs2[4:0] (arithmetic, sign-fills) |
| 1001–1111 | — | Unallocated |

For shift operations only bits `[4:0]` of rs2 are used as the shift amount. Bits `[31:5]` of rs2 are ignored at execution time.

**Pseudoinstructions:**

| Pseudo | Expands to | Notes |
|:---|:---|:---|
| `MOV rd, rs` | `ADD rd, rs, r0` | Copy register |
| `NEG rd, rs` | `SUB rd, r0, rs` | Two's complement negate |

---

## I-type

```
31      26 25    22 21    18 17    15 14             0
[ opcode ] [rd      ] [rs1    ] [funct  ] [imm          ]
    6           4          4        3           14
```

Two opcode values: one for immediate ALU operations, one for loads. The `funct` field selects the operation within each group. The immediate is 14 bits, always sign-extended to 32 bits before use. Range: −8192 to +8191.

### Immediate ALU (opcode ALU-I)

| funct | Mnemonic | Operation |
|:---|:---|:---|
| 000 | ADDI | rd = rs1 + sext(imm) |
| 001 | ANDI | rd = rs1 & sext(imm) |
| 010 | ORI  | rd = rs1 \| sext(imm) |
| 011 | XORI | rd = rs1 ^ sext(imm) |
| 100 | LSLI | rd = rs1 << imm[4:0] |
| 101 | LSRI | rd = rs1 >> imm[4:0] (logical) |
| 110 | ASRI | rd = rs1 >>> imm[4:0] (arithmetic) |
| 111 | —    | Unallocated |

For shift immediates only bits `[4:0]` of imm are used. Bits `[13:5]` must be zero in valid encodings; the assembler enforces this.

### Loads (opcode LOAD)

Effective address = rs1 + sext(imm).

| funct | Mnemonic | Operation | C equivalent |
|:---|:---|:---|:---|
| 000 | LB  | rd = sext(mem[EA][7:0])  | signed char |
| 001 | LBU | rd = zext(mem[EA][7:0])  | unsigned char |
| 010 | LH  | rd = sext(mem[EA][15:0]) | short |
| 011 | LHU | rd = zext(mem[EA][15:0]) | unsigned short |
| 100 | LW  | rd = mem[EA][31:0]       | int, pointer |
| 101–111 | — | Unallocated | |

LH/LHU require 2-byte aligned EA. LW requires 4-byte aligned EA. LB/LBU have no alignment requirement. Misaligned accesses produce architecturally undefined behaviour.

---

## S-type

```
31      26 25    22 21    18 17    15 14             0
[ opcode ] [rs2     ] [rs1    ] [funct  ] [imm          ]
    6           4          4        3           14
```

One opcode value for all stores. Effective address = rs1 + sext(imm). rs2 is the register whose value is written to memory. There is no rd field — stores produce no register result.

| funct | Mnemonic | Operation |
|:---|:---|:---|
| 000 | SB | mem[EA] = rs2[7:0] |
| 001 | SH | mem[EA] = rs2[15:0] |
| 010 | SW | mem[EA] = rs2[31:0] |
| 011–111 | — | Unallocated |

SH requires 2-byte aligned EA. SW requires 4-byte aligned EA. SB has no alignment requirement. Misaligned accesses produce architecturally undefined behaviour.

Sign is a property of interpretation on load, not on store. There are no SBU or SHU variants — the low bits of rs2 are always written as-is.

---

## B-type

```
31      26 25    22 21    18 17    15 14              0
[ opcode ] [rs1     ] [rs2    ] [cond   ] [offset        ]
    6           4          4        3           15
```

One opcode value for all conditional branches. rs1 and rs2 are compared according to `cond`. If the condition is true: PC = PC + sext(offset) << 2. If false: PC = PC + 4.

The offset is 15 bits, PC-relative, in units of 4 bytes. Effective byte range: ±128KB.

| cond | Mnemonic | Condition |
|:---|:---|:---|
| 000 | BEQ  | rs1 == rs2 (signed or unsigned, same bit pattern) |
| 001 | BNE  | rs1 != rs2 |
| 010 | BLT  | rs1 < rs2 (signed) |
| 011 | BGE  | rs1 >= rs2 (signed) |
| 100 | BLTU | rs1 < rs2 (unsigned) |
| 101 | BGEU | rs1 >= rs2 (unsigned) |
| 110–111 | — | Unallocated |

**Pseudoinstructions:**

| Pseudo | Expands to | Notes |
|:---|:---|:---|
| `BEQZ rs, offset` | `BEQ rs, r0, offset` | Branch if rs == 0 |
| `BNEZ rs, offset` | `BNE rs, r0, offset` | Branch if rs != 0 |
| `BGT rs1, rs2, offset` | `BLT rs2, rs1, offset` | Swap operands |
| `BLE rs1, rs2, offset` | `BGE rs2, rs1, offset` | Swap operands |

---

## U-type

```
31      26 25    22 21                              0
[ opcode ] [rd      ] [imm                            ]
    6           4                  22
```

One opcode value (LUI). Places the 22-bit immediate into bits `[31:14]` of rd, zeroing bits `[13:0]`. Used to construct arbitrary 32-bit constants in combination with ADDI.

### 32-bit constant construction

```
LUI  rd, upper(K)    ; rd[31:14] = upper22(K), rd[13:0] = 0
ADDI rd, rd, lower(K); rd = rd + sext(lower14(K))
```

**Sign adjustment:** ADDI sign-extends its 14-bit immediate. If bit 13 of the lower 14 bits of K is 1, ADDI treats the lower part as negative and subtracts. The assembler compensates by adding 1 to the LUI immediate. Formally: if `K[13] == 1` encode `upper = (K >> 14) + 1`, else `upper = K >> 14`. GCC's backend must implement this same adjustment when synthesising 32-bit constants in the `movsi` expand pattern.

---

## J-type

```
31      26 25    22 21                              0
[ opcode ] [rd      ] [offset                         ]
    6           4                  22
```

One opcode value (JAL). Unconditional PC-relative jump and link. rd receives PC+4 (the address of the instruction after JAL). Writing to r0 silently discards the return address. The offset is 22 bits, PC-relative, in units of 4 bytes. Effective byte range: ±8MB.

```
rd = PC + 4
PC = PC + sext(offset) << 2
```

Both operations are atomic — rd receives the return address before the jump takes effect.

**Pseudoinstructions:**

| Pseudo | Expands to | Notes |
|:---|:---|:---|
| `JMP offset` | `JAL r0, offset` | Discard return address |
| `CALL offset` | `JAL r14, offset` | Save return address in link register |

---

## JALR

```
31      26 25    22 21    18 17    15 14             0
[ opcode ] [rd      ] [rs1    ] [funct  ] [imm          ]
    6           4          4        3           14
```

Dedicated opcode. funct must be 000. Target address comes from a register — this is the indirect jump. rd receives PC+4. PC = rs1 + sext(imm).

```
target = rs1 + sext(imm)
rd     = PC + 4
PC     = target
```

Both operations are atomic.

**Pseudoinstructions:**

| Pseudo | Expands to | Notes |
|:---|:---|:---|
| `RET` | `JALR r0, r14, 0` | Return from function |
| `JR rs` | `JALR r0, rs, 0` | Indirect jump, discard return address |

---

## Opcode allocation

6-bit opcode field, 64 total slots. Concrete binary values are assigned here.

| Binary | Hex | Assigned to |
|:---|:---|:---|
| 000000 | 0x00 | R-type |
| 000001 | 0x01 | I-type immediate ALU |
| 000010 | 0x02 | I-type loads |
| 000011 | 0x03 | S-type stores |
| 000100 | 0x04 | B-type branches |
| 000101 | 0x05 | U-type LUI |
| 000110 | 0x06 | J-type JAL |
| 000111 | 0x07 | JALR |
| 001000–111111 | 0x08–0x3F | Unallocated |