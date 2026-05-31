# 8-Bit MIPS Processor — Logisim Simulation

> **CSE 210 · Computer Architecture Sessional** | Section A2 · Group 01
> BUET · December 2024

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture at a Glance](#architecture-at-a-glance)
3. [Components Deep Dive](#components-deep-dive)
4. [Instruction Set](#instruction-set)
5. [Instruction Encoding](#instruction-encoding)
6. [Control Unit & Microprogram](#control-unit--microprogram)
7. [Memory Organization](#memory-organization)
8. [ICs Used](#ics-used)
9. [Toolchain — Assembler & ROM Loader](#toolchain--assembler--rom-loader)
10. [How to Run & Test in Logisim](#how-to-run--test-in-logisim)
11. [Challenges & Lessons Learned](#challenges--lessons-learned)
12. [Team](#team)
13. [Repository Structure](#repository-structure)

---

## Project Overview

This project is a **fully functional 8-bit MIPS processor** designed from the ground up and simulated in [Logisim](http://www.cburch.com/logisim/). Every component — the datapath, the register file, the custom ALU, the microprogrammed control unit, the instruction memory, and the data memory — was individually designed, tested, and then wired together into a single working CPU.

The processor implements a custom subset of the MIPS ISA: **16 instructions** spanning arithmetic, logic, memory access, conditional branching, and unconditional jumps. Instructions are **20-bits wide** and operate on **8-bit data and addresses**. Each instruction completes in exactly **one clock cycle** (single-cycle datapath).

On top of the hardware design, we built a complete **software toolchain** — a C++ assembler that translates MIPS assembly into hex machine code, and a Python script that injects that hex directly into the Logisim circuit file's instruction ROM. This end-to-end flow lets you write MIPS assembly and watch it execute inside the circuit without any manual hex editing.

**Key design highlights:**
- Single-cycle, 8-bit MIPS datapath
- Microprogrammed Control Unit (opcode → control word via ROM lookup)
- Custom 8-bit ALU (add, sub, AND, OR, NOR, SLL, SRL)
- 7-register file ($zero, $t0–$t4, $sp) with 8-bit registers
- 256-byte data memory with full stack support
- 28 ICs total (ROM, RAM, 74150/74154/74157 MUX/DeMUX, 7408 AND, 7432 OR, 7404 NOT, 7483 Adder)
- Automated assembly-to-simulation pipeline

---

## Architecture at a Glance

```
  ┌─────────────────────────────────────────────────────────────────────┐
  │                         MIPS Processor                              │
  │                                                                     │
  │   ┌──────┐    ┌──────────────┐    ┌────────────┐   ┌────────────┐  │
  │   │  PC  │───►│ Instruction  │───►│  Control   │   │    ALU     │  │
  │   │(8bit)│    │ Memory (ROM) │    │   Unit     │   │  (8-bit)   │  │
  │   └──┬───┘    │  20-bit out  │    │(Microprg.) │   │ ALUop[2:0] │  │
  │      │        └──────────────┘    └─────┬──────┘   └─────┬──────┘  │
  │    PC+1                                 │                 │         │
  │      │          ┌──────────────┐        │  Control Sigs   │  ZF,F   │
  │      │          │ Register File│◄───────┴────────────────►│         │
  │      │          │ 7 Registers  │                           │         │
  │      │          │ (8-bit each) │                           │         │
  │      │          └──────┬───────┘    ┌─────────────┐        │         │
  │      │                 │            │ Data Memory │◄───────┘         │
  │      │                 └───────────►│  RAM 256B   │                  │
  │      └─────────────────────────────┤  (8-bit R/W)│                  │
  │                                    └─────────────┘                  │
  └─────────────────────────────────────────────────────────────────────┘
```

The datapath follows the classic single-cycle MIPS architecture from Patterson & Hennessy's *Computer Organization and Design* (5th Ed.), adapted carefully for an 8-bit word size and our group-specific opcode assignments.

---

## Components Deep Dive

### 1. Program Counter (PC)

The PC is an **8-bit register** that triggers on the **falling edge** of the clock signal. After each clock cycle it increments by 1, supplying the next instruction address to the Instruction Memory. For branch instructions the PC is overridden with a computed target address; for `j` (jump) it is loaded with the 8-bit target field from the instruction directly.

### 2. Register File

The register file is a bank of **7 registers**, each 8 bits wide:

| Register | Role | Initial Value |
|----------|------|--------------|
| `$zero` | Constant zero | `0x00` |
| `$t0` | General purpose | — |
| `$t1` | General purpose | — |
| `$t2` | General purpose | — |
| `$t3` | General purpose | — |
| `$t4` | General purpose | — |
| `$sp` | Stack pointer | `0xFF` |

The register file supports **two simultaneous reads** (READ DATA 1, READ DATA 2) and **one synchronous write** (on the rising clock edge, gated by REG WRITE). Register addresses are 3 bits wide, selected via 3-bit READ REGISTER 1, READ REGISTER 2, and WRITE REGISTER inputs. The **REG DST** control signal selects between the I-type destination field and the R-type destination field when deciding which register to write.

### 3. Arithmetic Logic Unit (ALU)

The ALU is a custom **8-bit combinational unit** that accepts two 8-bit operands (A, B) and a **3-bit ALUop** control code. It outputs an 8-bit result **F** and a **Zero Flag (ZF)** used by branch logic.

| ALUop | Operation |
|-------|-----------|
| `000` | NOR |
| `001` | Subtraction (A − B) |
| `010` | Pass-through / Jump |
| `011` | Subtraction with immediate |
| `100` | Branch / compare |
| `101` | Addition (A + B) |
| `110` | Store word |
| `111` | Load word / OR |

The **ALU SRC** control signal selects the second ALU operand — either READ DATA 2 from the register file (R-type) or the sign-extended/zero-extended immediate field (I-type). For shift instructions (SLL, SRL), the **SHIFT** control signal activates the shift path and the shift amount is taken from the immediate/shamt field.

### 4. Control Unit (Microprogrammed)

The Control Unit is the brain of the processor. Rather than implementing a combinational decoder with gates, we used a **microprogrammed approach**: the 4-bit opcode from the instruction indexes into a **Control Word ROM**, which outputs all the required control signals simultaneously.

Control signals generated:

| Signal | Width | Purpose |
|--------|-------|---------|
| `ALU OP 0/1/2` | 3-bit | Selects ALU operation |
| `JMP` | 1-bit | Enables unconditional jump |
| `BRANCH` | 1-bit | Enables conditional branch (beq) |
| `BRANCH_N` | 1-bit | Enables branch-if-not-equal (bneq) |
| `MEM WRITE` | 1-bit | Enables data memory write (sw) |
| `MEM READ` | 1-bit | Enables data memory read (lw) |
| `REG WRITE` | 1-bit | Enables register file write |
| `MEM 2 REG` | 1-bit | Selects memory output vs ALU output for writeback |
| `ALU SRC` | 1-bit | Selects register vs immediate as ALU input |
| `REG DST` | 1-bit | Selects destination register for R vs I type |
| `SHIFT` | 1-bit | Activates shift datapath |

The `BRANCH_N` bit is a custom addition beyond the standard textbook design, required to handle `bneq` separately from `beq`. The branch logic combines the Zero Flag from the ALU with the BRANCH and BRANCH_N signals using AND/OR/NOT gates to compute the actual PC override.

### 5. Data Memory

The data memory is a **synchronous RAM with 256 bytes** of storage. It is byte-addressable with an **8-bit address** and an **8-bit data width**. It serves dual purpose:
- **Main data storage** — for `lw` / `sw` instructions
- **Stack** — accessed via the `$sp` register (grows downward from `0xFF`)

Control signals MEM READ and MEM WRITE gate the read and write operations. The MEM 2 REG signal selects whether the register writeback receives the ALU result or the memory read data.

### 6. Instruction Memory

The Instruction Memory is a **ROM** indexed by the 8-bit PC. Each address holds a **20-bit instruction word**. The ROM is pre-loaded with machine code before simulation using the Python ROM-loading script.

---

## Instruction Set

Our group (Section A2, Group 01) was assigned the following opcode sequence. Opcodes are 4-bit values (0–15):

| ID | Instruction | Type | Opcode (binary) | Opcode (hex) | Category |
|----|-------------|------|-----------------|--------------|----------|
| A  | `add`  | R | `0101` | `5` | Arithmetic |
| B  | `addi` | I | `1100` | `C` | Arithmetic |
| C  | `sub`  | R | `0001` | `1` | Arithmetic |
| D  | `subi` | I | `0011` | `3` | Arithmetic |
| E  | `and`  | R | `1011` | `B` | Logic |
| F  | `andi` | I | `1000` | `8` | Logic |
| G  | `or`   | R | `1111` | `F` | Logic |
| H  | `ori`  | I | `1101` | `D` | Logic |
| I  | `sll`  | R | `1010` | `A` | Logic |
| J  | `srl`  | R | `1110` | `E` | Logic |
| K  | `nor`  | R | `0000` | `0` | Logic |
| L  | `sw`   | I | `0110` | `6` | Memory |
| M  | `lw`   | I | `0111` | `7` | Memory |
| N  | `beq`  | I | `1001` | `9` | Control (conditional) |
| O  | `bneq` | I | `0100` | `4` | Control (conditional) |
| P  | `j`    | J | `0010` | `2` | Control (unconditional) |

---

## Instruction Encoding

All instructions are **20 bits wide**. There are three instruction formats:

### R-Type (register-register operations)
```
 19      16 | 15     12 | 11      8 |  7      4 |  3      0
 ┌──────────┬───────────┬───────────┬───────────┬──────────┐
 │  Opcode  │  Src Reg1 │  Src Reg2 │  Dst Reg  │  Shamt   │
 │  4 bits  │   4 bits  │   4 bits  │   4 bits  │  4 bits  │
 └──────────┴───────────┴───────────┴───────────┴──────────┘
```
Used by: `add`, `sub`, `and`, `or`, `nor`, `sll`, `srl`

### I-Type (immediate / memory / branch operations)
```
 19      16 | 15     12 | 11      8 |  7                  0
 ┌──────────┬───────────┬───────────┬────────────────────────┐
 │  Opcode  │  Src Reg1 │  Src Reg2 │  Address / Immediate   │
 │  4 bits  │   4 bits  │   4 bits  │        8 bits          │
 └──────────┴───────────┴───────────┴────────────────────────┘
```
Used by: `addi`, `subi`, `andi`, `ori`, `sw`, `lw`, `beq`, `bneq`

### J-Type (unconditional jump)
```
 19      16 | 15               8 |  7      4 |  3      0
 ┌──────────┬────────────────────┬───────────┬──────────┐
 │  Opcode  │  Target Jump Addr  │     0     │    0     │
 │  4 bits  │       8 bits       │   4 bits  │  4 bits  │
 └──────────┴────────────────────┴───────────┴──────────┘
```
Used by: `j`

### Register Encoding

Registers are encoded as 4-bit values in instructions:

| Register | Encoding |
|----------|----------|
| `$t0` | `0x0` |
| `$t1` | `0x1` |
| `$t2` | `0x2` |
| `$t3` | `0x3` |
| `$t4` | `0x4` |
| `$zero` | `0x5` |
| `$sp` | `0x6` |

---

## Memory Organization

The processor handles three distinct memory spaces:

```
Instruction Memory (ROM)          Data Memory (RAM)
┌─────────────────────┐           ┌─────────────────────┐
│  Addr 0x00  instr0  │  PC─────► │  Addr 0xFF  ← $sp   │  Stack top (grows ↓)
│  Addr 0x01  instr1  │           │  Addr 0xFE          │
│     ...             │           │     ...             │
│  Addr 0xFF  instrN  │           │  Addr 0x00          │
└─────────────────────┘           └─────────────────────┘
   8-bit address                     8-bit address
   20-bit instruction                8-bit data
```

- **Instruction Memory**: ROM, 8-bit address space, 20-bit data width. Loaded once via the Python script before simulation.
- **Data Memory**: RAM, 256 bytes, 8-bit address and data. Supports simultaneous read (lw) and write (sw).
- **Stack**: Implemented within Data Memory. `$sp` starts at `0xFF` and decrements on push (`sw $tX, 0($sp)` + `subi $sp, $sp, 1`).

---

## ICs Used

The circuit uses a total of **28 ICs**:

| IC Category | Part Number | Count |
|-------------|------------|-------|
| ROM | — | 2 |
| RAM | — | 1 |
| MUX | 74150 | 2 |
| DeMUX | 74154 | 1 |
| Registers | — | 9 |
| MUX | 74157 | 7 |
| NOT | 7404 | 1 |
| AND | 7408 | 2 |
| OR | 7432 | 1 |
| Adder | 7483 | 2 |
| **Total** | | **28** |

The SLL and SRL shift instructions required adding **2 extra MUXes** beyond the baseline design to route the shift-path data correctly. The `bneq` instruction required a custom gate network (AND + OR + NOT) to invert and combine the Zero Flag with the branch control signals.

---

## Toolchain — Assembler & ROM Loader

Writing machine code by hand in hex for every test would be brutally error-prone, especially with group-specific non-standard opcodes. So we built a two-stage automated pipeline:

```
  MIPS_INPUT.txt           INSTRUCTION_CONVERTOR          hexdata.hex
  (assembly source)  ──►  (C++ assembler, compiled)  ──►  (hex machine code)
                                                               │
                                                               ▼
                                                    RomEditor_HEX_auto.py
                                                    (patches MIPS.circ ROM)
                                                               │
                                                               ▼
                                                         MIPS.circ
                                                    (ready to simulate)
```

### Stage 1: C++ Assembler (`INSTRUCTION_CONVERTOR.cpp`)

This program reads `MIPS_INPUT.txt` (standard MIPS assembly syntax), resolves label addresses, encodes each instruction into a 5-hex-digit machine word (20-bit, represented as 5 nibbles), and writes them space-separated into `hexdata.hex`.

Key features:
- Label resolution via a two-pass approach (labels stored in a `map<string, int>`)
- Supports `#` and `//` line comments
- Handles all three instruction formats (R, I, J) with format-specific field packing
- Branch offsets are computed as `label_address − (current_instruction + 1)` (PC-relative)
- Immediate values are sign-extended and truncated to the lower 2 hex digits (8 bits)

### Stage 2: Python ROM Loader (`RomEditor_HEX_auto.py`)

This script parses the `MIPS.circ` Logisim circuit file (which is XML under the hood), locates the specific Instruction Memory ROM component by its position attribute (`loc="(400,520)"`), and overwrites its `contents` field with the hex data from `hexdata.hex`. It then writes the updated XML back to `MIPS.circ` — so the next time you open the circuit in Logisim, the program is already loaded.

```python
# The key patch — setting ROM contents in Logisim's XML
j.text = f"addr/data: 8 20 \n{hex_data}"
```

The format `addr/data: 8 20` tells Logisim that the ROM has an 8-bit address bus and a 20-bit data bus.

### Writing Assembly

MIPS assembly syntax used in `MIPS_INPUT.txt`:

```asm
# Initialize stack pointer
addi $sp, $zero, 255

# Arithmetic
addi $t0, $zero, 10      # $t0 = 10
add  $t2, $t0, $t1       # $t2 = $t0 + $t1
subi $t0, $t0, 1         # $t0 = $t0 - 1

# Memory
sw   $t3, 0($sp)         # mem[$sp + 0] = $t3
lw   $t1, 0($sp)         # $t1 = mem[$sp + 0]

# Control flow
beq  $t0, $t1, label     # if $t0 == $t1 goto label
bneq $t0, $t1, label     # if $t0 != $t1 goto label
j    label               # unconditional jump

label:
    # ...
```

---

## How to Run & Test in Logisim

### Prerequisites

- **Logisim** (2.7.1 or compatible) installed
- **Python 3** installed
- **MinGW / g++** (to recompile the C++ assembler if needed)
- The `8BitAlu.jar` library (included in the Project Spec; needed by Logisim)

---

### Step 1 — Load the ALU Library into Logisim

The custom 8-bit ALU is provided as a JAR file (`8BitAlu.jar`).

1. Open Logisim.
2. Go to **Project → Load Library → JAR Library**.
3. Select `8BitAlu.jar` from the `Project Spec/Project/` folder.
4. Click OK. The ALU component will now be available.

---

### Step 2 — Write Your Assembly Program

Open `A2_Grp01_Simulation/MIPS_INPUT.txt` in any text editor and write your MIPS assembly program. Follow the syntax shown above. Use labels freely — the assembler resolves them.

**Example program (Fibonacci):**
```asm
addi $sp, $zero, 255
addi $t0, $zero, 10
addi $t1, $zero, 1
addi $t2, $zero, 1
loop:
add  $t3, $t1, $t2
add  $t1, $zero, $t2
add  $t2, $zero, $t3
addi $t0, $t0, -1
bneq $t0, $t1, loop
done:
j done
```

---

### Step 3 — Assemble to Machine Code

Navigate to `A2_Grp01_Simulation/` in your terminal and run the C++ assembler:

```bash
# On Windows (pre-compiled binary included)
./INSTRUCTION_CONVERTOR.exe

# Or recompile from source first
g++ -o INSTRUCTION_CONVERTOR INSTRUCTION_CONVERTOR.cpp
./INSTRUCTION_CONVERTOR
```

This reads `MIPS_INPUT.txt` and produces `hexdata.hex`.

---

### Step 4 — Load Machine Code into the Circuit

Still in `A2_Grp01_Simulation/`, run the Python ROM loader:

```bash
python RomEditor_HEX_auto.py
```

This patches `MIPS.circ` by injecting the hex machine code directly into the Instruction Memory ROM. You should see:
```
hexdata.hex generated successfully by the C++ program.
ROM contents updated in MIPS.circ.
MIPS.circ updated successfully.
```

---

### Step 5 — Open and Simulate in Logisim

1. Open `A2_Grp01_Simulation/MIPS.circ` in Logisim.
2. If prompted about missing libraries, point Logisim to `8BitAlu.jar`.
3. Go to **Simulate → Simulation Enabled** (or press `Ctrl+E`).
4. Set the clock speed via **Simulate → Tick Frequency** (start with 1 Hz to observe each cycle clearly).
5. Press **Simulate → Ticks Enabled** (or `Ctrl+K`) to start the clock.

---

### Step 6 — Observe Execution

Watch the circuit execute your program clock cycle by clock cycle:

- **PC register**: increments each cycle, shows which instruction is executing
- **Register File outputs**: probe the register output pins to see live register values
- **Data Memory**: click on the RAM component to inspect memory contents
- **ALU outputs**: observe the F (result) and ZF (zero flag) pins
- **Control Unit**: observe active control signals to verify correct decoding

To **step manually** through instructions without auto-clock:
- Use **Simulate → Step Simulation** (`Ctrl+T`) for single-cycle stepping

To **reset** the processor:
- Toggle the reset pin (connected to the PC register's CLR input) or use **Simulate → Reset Simulation** (`Ctrl+R`)

---

### Verifying a Test

After running a program, check register and memory values:

1. Click the **Register File** component → inspect all 7 register values
2. Click the **Data Memory (RAM)** component → inspect byte values at each address
3. For stack operations, look at addresses `0xFF`, `0xFE`, `0xFD`, ... downward

---

## Challenges & Lessons Learned

Building this processor from scratch was one of the most genuinely satisfying (and occasionally maddening) experiences of the course. Here's an honest account of what gave us the most trouble:

**Opcode assignment confusion.** Our group-specific opcodes are non-standard, and getting the C++ assembler to output the right hex encoding was the very first wall we hit. We made the deliberate decision to write the assembler *before* drawing any circuit — this turned out to be a great call, because it let us verify instruction encodings programmatically without ever needing to hand-count bits in a hex dump.

**RAM, ROM and registers in Logisim — first contact.** None of us had used Logisim's memory primitives before. The first few hours were spent just figuring out how the address/data pin widths map to actual memory cells, and why writes weren't sticking until we got the clock edge polarity right.

**Arithmetic results silently writing to the wrong register.** One of the nastiest bugs: immediate arithmetic instructions (`addi`, `subi`) were computing correctly but writing the result to the wrong register. The root cause was a subtle error in the ordering of fields in the instruction word — specifically, which nibble position carried the source register and which carried the destination. Once we traced the actual bit wires from the instruction memory to the register file write port, the bug was obvious. But finding it took hours.

**Making `bneq` work.** The standard Patterson & Hennessy datapath only has `beq`. Implementing `bneq` required an extra control bit (`BRANCH_N`) and a small gate network (AND + OR + NOT) to invert the Zero Flag path. Getting the logic exactly right — so that `beq` and `bneq` don't interfere with each other — required careful truth-table analysis.

**Branching and jumping in simulation.** Watching the PC actually jump to a label for the first time, after all the wiring and debugging, was genuinely exciting. It took meticulous per-module testing before integration to get there.

**SLL and SRL.** Shift instructions complicated the MUX routing significantly — we needed two extra 74157 MUXes to route shift-amount vs. register-source to the ALU correctly under the SHIFT control signal.

Through all of this, we developed a real intuitive grasp of how a CPU works at the transistor-and-wire level. The abstraction gap between "a line of C code" and "billions of transistors switching" collapsed completely. Every `add` instruction we write now, we can visualize exactly which wires carry which bits at each nanosecond of a clock cycle.

---

## Team

**Section A2 · Group 01 · CSE 210 · BUET · December 2024**

| Student ID | Name | Contributions |
|------------|------|--------------|
| 2105032 | Nawriz Ahmed Turjo | Instruction Memory design, ALU, optimization, whole-circuit integration, circuit debugging, Python ROM loader, code debugging |
| 2105047 | Himadri Gobinda Biswas | Control Unit, Register File, C++ opcode assignment code, circuit testing, circuit debugging, whole-circuit integration |
| 2105048 | Shams Hossain Simanto | Data Memory, ALU, Register File, C++ instruction generator, circuit testing, whole-circuit integration |

---

## Repository Structure

```
Assignment 3 MIPS Processor/
│
├── readme.md                          ← You are here
│
├── Project Spec/
│   └── Project/
│       ├── CSE210_Project.pdf         ← Original assignment specification
│       └── 8BitAlu.jar                ← Custom ALU library for Logisim
│
└── A2_Grp01_Submission/
    ├── A2_Grp01_Report.pdf            ← Full technical report
    │
    ├── A2_Grp01_Simulation/           ← Main simulation files
    │   ├── MIPS.circ                  ← Logisim circuit (open this)
    │   ├── MIPS_INPUT.txt             ← Assembly source (edit this)
    │   ├── INSTRUCTION_CONVERTOR.cpp  ← C++ assembler source
    │   ├── INSTRUCTION_CONVERTOR.exe  ← Compiled assembler (Windows)
    │   ├── RomEditor_HEX_auto.py      ← ROM loader (automated pipeline)
    │   ├── RomEditor_Turjo_manual.py  ← ROM loader (manual hex variant)
    │   └── hexdata.hex                ← Generated machine code (auto-updated)
    │
    └── A2_Grp01_Necessary_Content/    ← Standalone toolchain files
        ├── INSTRUCTION_CONVERTOR.cpp
        ├── INSTRUCTION_CONVERTOR.exe
        ├── MIPS_INPUT.txt
        ├── RomEditor_HEX_auto.py
        ├── RomEditor_Turjo_manual.py
        └── hexdata.hex
```

---

*Designed with logic gates and stubbornness. Debugged with oscilloscopes of the mind.*
