## Register operations
```asm
; R-type
add  rd, rs1, rs2
sub  rd, rs1, rs2
and  rd, rs1, rs2
or   rd, rs1, rs2
xor  rd, rs1, rs2
not  rd, rs1
lsl  rd, rs1, rs2
lsr  rd, rs1, rs2
asr  rd, rs1, rs2

; Pseudoinstructions
mov  rd, rs1          ; expands to: add rd, rs1, r0
neg  rd, rs1          ; expands to: sub rd, r0, rs1
```

## Immediate operations
```asm
addi  rd, rs1, imm
andi  rd, rs1, imm
ori   rd, rs1, imm
xori  rd, rs1, imm
lsli  rd, rs1, imm
lsri  rd, rs1, imm
asri  rd, rs1, imm
```

## Loads
```asm
lb   rd, imm(rs1)     ; signed byte
lbu  rd, imm(rs1)     ; unsigned byte
lh   rd, imm(rs1)     ; signed halfword
lhu  rd, imm(rs1)     ; unsigned halfword
lw   rd, imm(rs1)     ; word
```

## Stores
```asm
sb   rs2, imm(rs1)    ; byte
sh   rs2, imm(rs1)    ; halfword
sw   rs2, imm(rs1)    ; word
```

## Branches
```asm
beq  rs1, rs2, label
bne  rs1, rs2, label
blt  rs1, rs2, label
bge  rs1, rs2, label
bltu rs1, rs2, label
bgeu rs1, rs2, label

; Pseudoinstructions
beqz rs1, label       ; expands to: beq rs1, r0, label
bnez rs1, label       ; expands to: bne rs1, r0, label
bgt  rs1, rs2, label  ; expands to: blt rs2, rs1, label
ble  rs1, rs2, label  ; expands to: bge rs2, rs1, label
```

## Jump and link
```asm
jal  rd, label        ; PC-relative jump, rd = PC+4
jalr rd, rs1, imm     ; indirect jump, rd = PC+4

; Pseudoinstructions
jmp  label            ; expands to: jal r0, label
call label            ; expands to: jal r14, label
ret                   ; expands to: jalr r0, r14, 0
jr   rs1              ; expands to: jalr r0, rs1, 0
```

## Upper immediate
```asm
lui  rd, imm          ; rd[31:14] = imm, rd[13:0] = 0
```

## Comments and conventions
```asm
; semicolon for comments
; registers named r0-r15
; immediates are decimal by default, 0x prefix for hex
; labels followed by colon: loop:
; memory operand format: offset(base) e.g. 4(r13)
```
