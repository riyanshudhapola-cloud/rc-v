# P23 Microprogrammed control (control ROM) - dossier

Answer in your own words; short answers are fine. Diagrams can be photos of
hand drawings (put them next to this file and link them).

## U1 Digital logic
The circuit is purely combinational: one ROM (a 5-to-32 decoder plus OR gates), no flip-flops, because the output depends only on the current opcode bits.

I chose the control bit memwrite (ctrl bit 9). It is 1 only for STORE, whose op_hi is 01000 (address 8).

| op_hi (a4 a3 a2 a1 a0) | Instruction class | memwrite |
|---|---|---|
| 01000 | STORE | 1 |
| 00000 | LOAD | 0 |
| 00100 | OP-IMM | 0 |
| 00101 | AUIPC | 0 |
| 01100 | OP | 0 |
| 01101 | LUI | 0 |
| 11000 | BRANCH | 0 |
| 11001 | JALR | 0 |
| 11011 | JAL | 0 |
| 11100 | SYSTEM | 0 |
| all other addresses | unused | 0 |

Simplified expression: memwrite = a4' . a3 . a2' . a1' . a0'
(a single 5-input AND gate with inverted inputs on a4, a2, a1, a0). No simplification is possible because only one of the 32 input combinations gives 1. In the same way, memread (bit 10) = a4' . a3' . a2' . a1' . a0' (LOAD only).

## U2 Digital components
The Digital circuit uses a single ROM with 5 address bits and 16 data bits (32 words of 16 bits). The input op_hi (5 bits) is wired to the ROM address input and the ROM data output is wired to the output ctrl (16 bits). Inside, a ROM is a decoder (selects one of 32 words) followed by OR gates for each output bit. The circuit has no adders, multiplexers or registers, because it only looks up a stored value. Unused addresses hold 0.

## U3 Data representation
| Instruction class | op_hi binary | op_hi decimal | ctrl hex | ctrl binary | ctrl decimal |
|---|---|---|---|---|---|
| LUI | 01101 | 13 | 5830 | 0101 1000 0011 0000 | 22576 |
| LOAD | 00000 | 0 | 5480 | 0101 0100 1000 0000 | 21632 |
| STORE | 01000 | 8 | 1210 | 0001 0010 0001 0000 | 4624 |

Reading the bits: LUI has regwrite (bit 14), bsel_imm (bit 12), a_zero (bit 11) and immsel = 011 (U-type) set. LOAD has regwrite, bsel_imm, memread (bit 10) and wbsel = 01 (memory). STORE has bsel_imm, memwrite (bit 9) and immsel = 001 (S-type), and no regwrite.

Overflow and sign extension: this block does no arithmetic, so overflow cannot happen, and bit 15 is always 0, so the signed and unsigned values of ctrl are the same. The errors that can happen are address errors: using all 7 opcode bits instead of opcode[6:2], or entering a word at the wrong address. Sign extension of immediates is done in other blocks (P18, P19); this block only selects the type through immsel.

## U4 Computer organisation
The block works in the decode step of the instruction cycle: the opcode field of the fetched instruction is turned into the 16-bit control word. Every instruction needs it, because every instruction (LUI, AUIPC, JAL, JALR, branches, loads, stores, ALU immediate, ALU register, SYSTEM) needs its control signals. The control word is then used in the execute, memory, write-back and next-PC steps.

## U5 Processors
In RC-V, microprogramming is a single lookup in a control ROM, because the processor is single-cycle. A CISC processor uses microprogrammed control much more heavily: instructions are complex and take many steps, so the control store is large and a micro-program counter steps through several micro-instructions for one instruction. A stack machine needs much less decoding, because operands are implicit (top of the stack) and there are no register fields, so its control logic is simpler. A GPU uses one control unit to drive many parallel lanes (SIMD), as in the four-lane vector unit of P51, instead of decoding each lane separately. In Mano's Basic Computer, control is hardwired from the decoder outputs D0-D7 and the timing signals T0-T6; RC-V replaces the timing signals with one clock per instruction and can use either hardwired (P22) or microprogrammed (P23) control.

## U6 Memory and I/O
My block does not access memory or I/O itself. It produces the control signals that tell other blocks to do so: memread (bit 10), memwrite (bit 9) and wbsel (bits 8-7, where 1 selects memory data for write-back). The ALU computes the address, the address decoder (P31) and bus arbiter (P34) route it to RAM (P29) or the I/O bus (P33), the read selector (P32) chooses the data source, and the load extender (P28) formats the data before write-back. I/O is memory-mapped, so I/O transfers are ordinary loads and stores.

## E1 Instruction trace: SRLI
Example instruction: srli a0, a1, 2 (shift register a1 right by 2 bits, logical, result into a0). Encoding: 0000000 00010 01011 101 01010 0010011 = 0x0025D513. Assume a1 = 0x00000010 and PC = 0x00000100.

| Step | Block | What happens | Main signal value |
|---|---|---|---|
| 1 | P14 PC, P30 ROM | The PC addresses the ROM, which returns the instruction | PC = 0x00000100, instr = 0x0025D513 |
| 2 | P03 PC incrementer | Computes the next sequential address | PC + 4 = 0x00000104 |
| 3 | P17 field splitter | Splits the instruction into fields | opcode = 0010011, rd = 10 (a0), funct3 = 101, rs1 = 11 (a1), shift amount = 2 |
| 4 | P21 class decoder | Recognises the OP-IMM class | class = OP-IMM |
| 5 | P22 / P23 / P24 | The hardwired unit and my control ROM both produce the control word; P24 selects one and checks they agree. My ROM is addressed by op_hi = 00100 | ctrl = 0x5000: regwrite = 1, bsel_imm = 1, immsel = 0 (I), wbsel = 0 (ALU) |
| 6 | P18 / P20 immediate | Builds the I-type immediate, which holds the shift amount | imm = 2 |
| 7 | P13 register read | Reads register a1 | rs1 value = 0x00000010 |
| 8 | P26 operand select | A = rs1 value, B = immediate (bsel_imm = 1) | A = 0x00000010, B = 2 |
| 9 | P10 ALU control | funct3 = 101 and instruction bit 30 = 0 select shift right logical (bit 30 = 1 would be SRAI) | operation = SRL |
| 10 | P06 shifter, P08 result select | The barrel shifter shifts A right by B, and the result selector passes it on | result = 0x00000004 |
| 11 | P25 write-back, P11 / P12 | wbsel = 0 selects the ALU result, and regwrite = 1 writes it into a0 | a0 = 0x00000004 |
| 12 | P16 / P15 next PC | No branch or jump, so the next PC is PC + 4 | next PC = 0x00000104 |

The memory blocks are not used (memread = 0, memwrite = 0).

## E2 Neighbour review
P22 (Hardwired control unit) review: <paste link to your review here>
P24 (Control selector and checker) review: <paste link to your review here>

## E3 Where my block sits
My block is in the control layer. It is fed by the instruction field splitter (P17), which supplies the opcode bits op_hi = opcode[6:2]. Its output, the 16-bit control word, goes to P24, which chooses between it and the hardwired control word of P22 and raises a mismatch flag if they differ. The selected control word then controls blocks such as the write-back selector (P25), the operand selector (P26), the immediate selector (P20), the next-PC logic (P15, P16) and the memory blocks.

## E4 Syllabus map
- [x] Logic gates, Boolean algebra, K-maps
- [x] Combinational circuits (adders, decoders, encoders, multiplexers)
- [ ] Flip-flops, registers, counters
- [x] Memory (RAM, ROM)
- [x] Number systems, complements, signed arithmetic
- [x] Bus organisation, instruction codes and formats
- [x] Instruction cycle, hardwired vs microprogrammed control
- [ ] Interrupts and the interrupt cycle
- [ ] Pipelining
- [ ] Register organisation, stack, addressing modes
- [x] RISC vs CISC, multicore, GPU
- [ ] Memory hierarchy, cache
- [ ] I/O interface, programmed / interrupt-driven I/O, DMA
