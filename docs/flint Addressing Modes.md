# flint Addressing Modes
flint has 5 addressing modes. While this may seem a bit strange, flint has 5 because it properly exercises the GCC addressing mode subsystem. RISC-V has a single memory addressing mode: base register + 12-bit signed immediate offset and AArch64 has pre/post-increment modes, so flint finds a middle ground between them.

## Addressing modes
1. **Base + immediate offset:**
2. **Base + register offset:**
3. **Pre-increment:**
4. **Post-increment:**
5. **PC-relative:**
```
1. [rs1 + imm]       — base register, constant offset
2. [rs1 + rs2]       — base register, offset register
3. [rs1 + imm]!      — base register, constant offset, writeback
4. [rs1], imm        — base register, constant offset, writeback
5. [PC + imm]        — constant offset only, base is implicit
```