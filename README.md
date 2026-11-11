From RV32I to Pipelined, Cached RV32IM: Eight Chisel Stages
11 nov. 2026 · @bako
Overview
Eight Chisel projects take one RISC-V core from 19 ALU instructions to a pipelined RV32IM with an I-cache and a D-cache, each stage adding one idea to a design that already passes its tests. Every stage keeps the same package (rv32i), the same interface style (instruction port, data port, per-cycle retirement trace) and the same reference model, so each step changes only the hardware it is about.
All eight share one toolchain and one project kit:
• Toolchain: Scala 2.13.12, Chisel 3.6.1 with its compiler plugin, chiseltest 0.6.2 on Treadle (pure Scala, no simulator to install), sbt 1.10.2, Java 11 or newer.
• Sources: one Scala file per hardware block (ALU, decoder, immediate generator, register file, CSR file, multiplier, divider, caches), a core, and a test wrapper that adds memories loaded from program.hex / data.hex.
• Tests: unit tests per block, lockstep program tests against an instruction-level RefModel, timing tests, waveform demos with ready-made GTKWave sessions (core.gtkw, assembly translate filters), and planted-bug checks that prove the tests catch real faults.
• Output: Emit writes SystemVerilog through CIRCT firtool when installed, otherwise through Chisel 3's built-in compiler; Stage 1 also writes one Verilog file per module with a Verilog testbench for each.
Stages 1–5: single-cycle cores
The first five stages grow the instruction set on a single-cycle datapath: every instruction completes in one clock, except a division.
1. Stage 1, RV32I ALU core (rv32i-stage1). The 10 R-type and 9 I-type ALU instructions (ADD … AND, ADDI … SRAI). Blocks: ALU with 10 operations, decoder, sign-extended I immediate, 32 × 32 register file with x0 hard-wired to zero. The pc always advances by 4; any other word runs as a no-op with valid = 0.
2. Stage 2, loads and JALR (rv32i-stage2). Adds a read-only data memory, the five loads (LB, LH, LW, LBU, LHU) with byte and half-word alignment, and JALR, the first instruction that changes the pc.
3. Stage 3, complete RV32I + Zicsr (rv32i-stage3). Stores with byte strobes, the six branches, LUI, AUIPC and JAL, the six CSR instructions, and precise machine-mode traps: illegal instruction, ECALL, EBREAK and misaligned fetch, load and store, with mtvec, mepc, mcause, mtval and MRET.
4. Stage 4, RV32IM multiplication (rv32im-stage4). A combinational 33 × 33 multiplier executes MUL, MULH, MULHSU and MULHU in the same single cycle; division is still illegal.
5. Stage 5, RV32IM complete (rv32im-stage5). A sequential divider (one quotient bit per cycle) adds DIV, DIVU, REM and REMU: 34 cycles, or 2 for divide-by-zero and signed overflow, with the RISC-V-defined results. Only division is sequential; every other instruction still takes one cycle, and misa reads 0x40001100.
Stages 6–8: pipelines and caches
With the ISA complete, the last three stages keep RV32IM + Zicsr unchanged and rebuild the microarchitecture for speed: the reference model and every program test carry over as they are.
6. Stage 6, 2-stage pipeline (rv32im-stage6). IF fetches at fetchPc into the IfEx register (valid, pc, inst); EX does everything else. Fetch assumes pc + 4; a taken branch, jump, trap or MRET redirects fetch and flushes IfEx, costing one bubble. During a division IF and EX hold for 33 cycles.
7. Stage 7, 5-stage pipeline (rv32im-pipe5). IF, ID, EX, MEM, WB with registers IfId, IdEx, ExMem, MemWb. Forwarding from MEM and WB plus a WB→ID bypass remove most stalls; a load-use pair costs 1 cycle, a redirect (resolved in EX) 2, and the pipeline fill 4. Exceptions stay precise: they are taken in EX, older instructions drain, younger ones are flushed.
8. Stage 8, 5-stage pipeline with caches (rv32im-stage8). A 1 KiB direct-mapped I-cache and a 1 KiB direct-mapped write-through, no-write-allocate D-cache (64 lines × 16 bytes each) sit in front of main memories with a 4-cycle latency. While either cache is busy the whole pipeline freezes; the divider keeps working and holds its result. An advance signal from the core makes each store write through exactly once, even when the I-cache freezes the pipeline in the cycle the D-cache acknowledges the store.
The eight stages side by side
Each row adds one thing to the row above it; the cycle costs are those the timing tests check exactly.
Stage
Project
ISA
New hardware
Cycle cost
Tests
1
rv32i-stage1
19 ALU instructions
ALU, decoder, immediate, register file
1 per instruction
32
2
rv32i-stage2
+ 5 loads, JALR
Read-only data memory, load alignment
1
33
3
rv32i-stage3
RV32I + Zicsr
Stores, branches, jumps, CSR file, precise traps
1
30
4
rv32im-stage4
+ MUL, MULH, MULHSU, MULHU
Combinational 33 × 33 multiplier
1
34
5
rv32im-stage5
RV32IM + Zicsr
Sequential divider
1; DIV/REM 34
47
6
rv32im-stage6
same
2-stage pipeline IF | EX
+ 1 bubble per redirect; DIV/REM + 33
48
7
rv32im-pipe5
same
5-stage pipeline, forwarding, load-use stall
fill 4; load-use + 1; redirect + 2; DIV/REM + 33
37
8
rv32im-stage8
same
I-cache, D-cache, main memories, freeze
+ refill (≈ 6) per miss, + ≈ 5 per store
52
Test counts are those of the last full sbt test run of each project. Stage 1's 32 include the 7 suites that write vectors for its Verilog testbenches.
How every stage is verified
The same four-layer method runs in every project, so a stage passes only if its new hardware is right and nothing older broke.
• Unit tests check each block against Scala expectations: all ALU operations on edge values and thousands of random operands, every immediate, every divider corner case, and, in Stage 8, each cache against a directory model behind a randomly slow memory.
• Lockstep program tests run the core next to RefModel and compare the trace every cycle: pc, next pc, instruction, register written, memory access, trap and cause. Directed programs (loops, dependency chains, all six exception types) are joined by 3,000-instruction random programs; Stage 8 adds random busy cycles on both cache ports.
• Timing tests fix the cycle cost of each case: one instruction per cycle, the divider's 33 extra cycles, pipeline fill, bubbles per redirect and load-use, and the exact cost of a cache freeze.
• Planted bugs prove the tests have teeth: faults such as SRA decoded as SRL, a register write during a cache freeze, a cache hit that ignores the tag, or a store written twice must each make the suite fail.
Waveform demos write VCD files with a GTKWave session per demo, grouped by stage, with registers in decimal and instruction words shown as assembly. For hardware flows, Emit produces SystemVerilog (or plain Verilog with the legacy compiler), and Stage 1's EmitSplit writes one file per module, each with its own self-checking Verilog testbench for Icarus Verilog or Verilator.
Lessons and next steps
The series shows that a core can grow from 19 instructions to a cached pipeline without ever leaving a known-good state, because the reference model and the trace contract stay fixed while the hardware changes.
• Keep the trace contract stable. One retirement record per cycle let the same lockstep harness check a single-cycle core, two pipelines and a cached system.
• Stalls hide in the interactions. The real bugs came from two mechanisms meeting: a division during a freeze, or a D-cache store acknowledged while the I-cache still froze the pipeline.
• Test the tests. Planted bugs found test gaps that passing suites did not show.
Natural next steps: branch prediction to cut the redirect cost, a write buffer to hide store latency, a set-associative or write-back D-cache, interrupts (mie/mip and a timer), and FENCE.I with I-cache invalidation.
