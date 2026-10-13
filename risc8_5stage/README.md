# 8-bit RISC Processor Core — 5-Stage Pipeline (RTL Design)

A synthesizable 8-bit RISC CPU in Verilog-2001 with a classic 5-stage
pipeline: **IF → ID → EX → MEM → WB**.

This is a custom 8-bit RISC instruction set, not RISC-V. RISC-V's smallest base is 32-bit.

## Specification

| Item | Value |
|---|---|
| Data width | 8 bits |
| Instruction width | 16 bits, fixed |
| Registers | 8 × 8-bit (R0–R7) |
| Memory | Harvard: 256 × 16 instruction ROM, 256 × 8 data RAM |
| Flags | Z (zero), C (carry / borrow) |
| Instructions | 16 (NOP ADD SUB AND OR XOR SHL SHR LDI ADDI LD ST JMP BZ BNZ HLT) |
| Pipeline | IF, ID, EX, MEM, WB |
| Forwarding | EX/MEM → EX, MEM/WB → EX, WB → ID (register-file bypass) |
| Load-use hazard | 1-cycle stall |
| Branches | Resolved in EX, 2-cycle penalty when taken |

## Folder structure and who calls whom

```
risc8_5stage/
├── run_sim.bat / run_sim.sh   one-click simulation
├── files.f                   Verilog file list for iverilog -c
├── rtl/
│   ├── risc8_5stage_core.v   TOP: pipeline registers, PC, branch, halt
│   │     ├── calls imem          (rtl/imem.v)          Stage 1 IF
│   │     ├── calls control_unit  (rtl/control_unit.v)  Stage 2 ID
│   │     ├── calls regfile       (rtl/regfile.v)       ID read / WB write
│   │     ├── calls hazard_unit   (rtl/hazard_unit.v)   forwarding + stall
│   │     ├── calls alu           (rtl/alu.v)           Stage 3 EX
│   │     └── calls dmem          (rtl/dmem.v)          Stage 4 MEM
│   ├── alu.v, regfile.v, imem.v, dmem.v, control_unit.v, hazard_unit.v
├── tb/
│   ├── tb_risc8_5stage.v     directed self-checking test (calls risc8_5stage_core)
│   └── tb_dump.v             dumps final state for random testing
├── sw/
│   ├── program.s             test program (assembly)
│   ├── asm8.py               assembler: program.s -> program.hex
│   ├── iss.py                golden reference model (no pipeline)
│   └── random_test.py        random programs: RTL vs iss.py
└── program.hex               machine code loaded by imem.v ($readmemh)
```

## Quick start

- **Windows:** install Python 3 and Icarus Verilog (both added to PATH), then double-click `run_sim.bat`.
- **Linux / macOS:** run `sh run_sim.sh`.

You should see `*** ALL TESTS PASSED ***`. The expected full output is in `sim_log.txt`.

## How to run (manual)

```bash
# 1. Directed test
python3 sw/asm8.py sw/program.s program.hex
iverilog -g2005 -o sim.vvp -c files.f
vvp sim.vvp
gtkwave risc8_5stage.vcd

# 2. Random regression (500 programs vs golden model)
python3 sw/random_test.py 500
```

Note: `random_test.py` overwrites `program.hex`. Re-run step 1's assembler afterwards.

## Results

| Metric | 3-stage | 5-stage |
|---|---|---|
| Directed test | All pass | All pass |
| Random programs vs golden model | – | 500 / 500 match |
| Cycles for test program (34 instr.) | 41 | 51 |
| CPI | 1.21 | 1.50 |
| Load-use stalls | 0 | 1 |
| Taken-branch penalty | 1 cycle | 2 cycles |
| iCE40 HX8K logic cells | 4,418 | 547 |
| Block RAMs | 1 | 2 |
| Fmax (iCE40 HX8K, nextpnr) | 48.8 MHz | 73.2 MHz |

The 5-stage core needs more cycles per instruction, but its clock is 1.5x faster. Execution time for the test program is about 697 ns, against 840 ns for the 3-stage core.

Its data-memory address comes straight from the EX/MEM register, so synthesis maps the data memory into block RAM. In the 3-stage design that memory had to be built from 2,048 flip-flops.

## Error check

| Check | Tool | Result |
|---|---|---|
| Lint, all RTL | Verilator 5.020 `-Wall` | 0 errors, 0 warnings |
| Lint, both testbenches | Verilator 5.020 `-Wall --timing` | 0 errors, 0 warnings |
| Compile and simulate | Icarus Verilog 12 `-Wall` | 0 errors, 0 warnings |
| Synthesis | Yosys `synth_ice40` + `check -assert` | 0 errors, 0 warnings, no latches |
| Python scripts | `py_compile` | OK |
