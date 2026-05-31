# 4-Bit ALU — Design, Simulation & Hardware Implementation

> **CSE 210 · Computer Architecture Sessional** | Section A2 · Group 02
> BUET · September–October 2024

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Assigned Operations](#assigned-operations)
3. [Architecture & Design Approach](#architecture--design-approach)
4. [Datapath — How Every Operation Flows Through the Adder](#datapath--how-every-operation-flows-through-the-adder)
5. [Boolean Minimization — K-Map Derivations](#boolean-minimization--k-map-derivations)
6. [Truth Table for Intermediate Signals](#truth-table-for-intermediate-signals)
7. [Status Flags](#status-flags)
8. [Block Diagram](#block-diagram)
9. [ICs Used](#ics-used)
10. [Optimization — How We Got to 12 ICs](#optimization--how-we-got-to-12-ics)
11. [Simulation](#simulation)
12. [Hardware Implementation](#hardware-implementation)
13. [Challenges & The Moment It All Worked](#challenges--the-moment-it-all-worked)
14. [Team](#team)
15. [Repository Structure](#repository-structure)

---

## Project Overview

This project is a **4-bit Arithmetic Logic Unit (ALU)** designed entirely from the gate level up — first verified in software simulation (Logisim), then built in physical hardware on breadboards using real TTL ICs. The requirement was both demanding and thrilling: implement a full ALU capable of 6 distinct operations, generate all four standard processor status flags (Carry, Overflow, Sign, Zero), and do it with the **minimum possible number of ICs**.

The design is **purely combinational** — no clock, no state, just wires, gates, and adder carry chains computing results instantaneously from the 4-bit A and B inputs and a 3-bit control word. Every single logic gate in the final hardware traces back to a deliberate choice made during K-map minimization.

**Key highlights:**
- 6 operations (3 arithmetic, 3 logical) selected by a 3-bit control signal (cs2, cs1, cs0)
- All operations unified through a dual-adder core — arithmetic and logic alike
- Full flag set: Carry (CF), Overflow (VF), Sign (SF), Zero (ZF)
- Only **12 ICs** in the final hardware implementation
- Dual simulation + physical hardware delivery

---

## Assigned Operations

Our group (Section A2, Group 02) was assigned the following operation set:

| cs2 | cs1 | cs0 | Operation | Description |
|:---:|:---:|:---:|-----------|-------------|
| 0 | 0 | 0 | **Add** | A + B |
| 0 | X | 1 | **Sub** | A − B (two's complement subtraction) |
| 0 | 1 | 0 | **AND** | A ∧ B (bitwise AND) |
| 1 | 0 | 0 | **XOR** | A ⊕ B (bitwise XOR) |
| 1 | X | 1 | **Complement A** | Ā (bitwise NOT of A) |
| 1 | 1 | 0 | **NEG A** | −A (two's complement negation = Ā + 1) |

> `X` denotes a don't-care: both `cs1=0` and `cs1=1` produce the same function for that row.

That's **3 arithmetic operations** (Add, Sub, NEG A) and **3 logical operations** (AND, XOR, Complement A).

---

## Architecture & Design Approach

The central insight that shapes the entire design: **every operation can be expressed as `X + Y + C_in`**, where X and Y are carefully pre-processed versions of the inputs A and B, and C_in is the carry-in to the adder. By routing different expressions through the same two 4-bit adders, a single arithmetic datapath handles all six operations.

```
         ┌──────────────────────────┐
         │    CONTROL LOGIC         │
  cs2 ──►│  S₁, S₂, S₃, en₃, C_in  │
  cs1 ──►│  (from K-map expressions)│
  cs0 ──►└────────┬─────────────────┘
                  │
    ┌─────────────▼──────────────┐
    │  Dual 4×1 MUX (74153×2)   │ ◄── selects X_i from {A_i, A_i∧B_i, A_i⊕B_i, Ā_i}
    │  Select: S₂, S₁           │
    └─────────────┬──────────────┘
                  │ X₃ X₂ X₁ X₀
    ┌─────────────▼──────────────┐     ┌─────────────────────────┐
    │   4-bit Adder #1  (7483)   │ ◄── │ 2×1 MUX (74157) Y₀–Y₂  │
    │   X₀+Y₀, X₁+Y₁, X₂+Y₂    │     │ selects Y from {B, B̄, 0}│
    │   → produces C₃ (carry 3) │     └─────────────────────────┘
    └─────────────┬──────────────┘
           C₃     │ S₃ S₂ S₁
    ┌─────────────▼──────────────┐
    │   4-bit Adder #2  (7483)   │ ◄── X₃+Y₃ + C_in(=C₃)
    │   → produces S₀(MSB), C_out│
    └────────┬────────┬──────────┘
             │        │
           C_out     S₀–S₃
         (= CF)    (result bits)
             │
     ┌───────▼──────────────────┐
     │  Flag Logic              │
     │  CF=C_out  SF=S₀         │
     │  VF=C₃⊕C_out  ZF=NOR(…) │
     └──────────────────────────┘
```

The two-adder split is not arbitrary — it is required to extract the internal carry `C₃` (the carry out of bit position 2 into bit position 3), which is otherwise buried inside a single 7483 IC and inaccessible. `C₃` is needed alongside `C_out` to compute the **Overflow Flag** as `VF = C₃ ⊕ C_out`.

---

## Datapath — How Every Operation Flows Through the Adder

The genius of a unified adder-based ALU is that once you understand what `X_i`, `Y_i`, and `C_in` need to be for each operation, the hardware practically writes itself.

### First Adder Input — X_i

`X_i` has four possible values depending on the operation:

| Value | Operations that need it |
|-------|------------------------|
| `A_i` | Add, Sub (pass A straight through) |
| `A_i ∧ B_i` | AND (pre-compute AND, add to 0) |
| `A_i ⊕ B_i` | XOR (pre-compute XOR, add to 0) |
| `Ā_i` | Complement A, NEG A (inverted A, added to 0 or 1) |

Four options → a **dual 4×1 MUX** (one 74153 per bit, two ICs for all 4 bits) with selection bits **S₁** and **S₂**.

### Second Adder Input — Y_i

`Y_i` has three possible values:

| Value | Operations that need it |
|-------|------------------------|
| `B_i` | Add (plain addition) |
| `B̄_i` | Sub (A + B̄ + 1 = A − B by two's complement) |
| `0` | AND, XOR, Complement A, NEG A (logical ops just pass X_i through) |

Three options → a **2×1 MUX** (74157) with enable `en₃`. When `en₃=1`, the MUX output is forced to 0. When `en₃=0`, selection bit **S₃** chooses between `B_i` and `B̄_i`.

### Carry Input — C_in

`C_in = 1` for subtraction (to complete the two's complement: A + B̄ + 1) and for NEG A (Ā + 1). For all other operations `C_in = 0`.

---

## Boolean Minimization — K-Map Derivations

Five intermediate control signals were derived via Karnaugh map minimization from the three control inputs (cs2, cs1, cs0):

### S₁ — First selection bit for the 4×1 X-MUX

Minterms at: cs2=0,cs1=1,cs0=0 and cs2=1,cs0=0 and cs2=1,cs0=1

```
        cs1·cs0
cs2  |  00  01  11  10
  0  |   0   0   0   1
  1  |   0   1   1   1
```

**S₁ = cs2·cs0 + cs1·cs0̄**

### S₂ — Second selection bit for the 4×1 X-MUX

Active whenever cs2=1 (selects complement/XOR path):

```
S₂ = cs2
```

### S₃ — Selection bit for the 2×1 Y-MUX

Active whenever cs0=1 (selects B̄ for subtraction operations):

```
S₃ = cs0
```

### en₃ — Enable (zeroing) signal for Y-MUX

Forces Y_i=0 for all logical operations and NEG A:

```
en₃ = cs2 + cs1·cs0̄
```

### C_in — Carry input to the adder chain

Must be 1 for Sub (cs0=1, cs2=0) and NEG A (cs2=1, cs1=1, cs0=0):

```
C_in = cs2̄·cs0 + cs2·cs1·cs0̄
```

---

## Truth Table for Intermediate Signals

| cs2 | cs1 | cs0 | Function | X_i | S₂ | S₁ | Y_i | S₃ | en₃ | C_in |
|:---:|:---:|:---:|----------|-----|:--:|:--:|-----|:--:|:---:|:----:|
| 0 | 0 | 0 | Add | A_i | 0 | 0 | B_i | 0 | 0 | 0 |
| 0 | 0 | 1 | Sub | A_i | 0 | 0 | B̄_i | 1 | 0 | 1 |
| 0 | 1 | 0 | AND | A_i ∧ B_i | 0 | 1 | 0 | X | 1 | 0 |
| 0 | 1 | 1 | Sub | A_i | 0 | 0 | B̄_i | 1 | 0 | 1 |
| 1 | 0 | 0 | XOR | A_i ⊕ B_i | 1 | 0 | 0 | X | 1 | 0 |
| 1 | 0 | 1 | Complement A | Ā_i | 1 | 1 | 0 | X | 1 | 0 |
| 1 | 1 | 0 | NEG A | Ā_i | 1 | 1 | 0 | X | 1 | 1 |
| 1 | 1 | 1 | Complement A | Ā_i | 1 | 1 | 0 | X | 1 | 0 |

---

## Status Flags

All four x86-style processor flags are generated combinationally from the adder outputs:

### Carry Flag (CF)
**CF = C_out** — the final carry out of the second 7483 adder. Set to 1 if the unsigned addition overflowed the 4-bit result. Cleared to 0 for all logical operations (AND, XOR, Complement A).

### Overflow Flag (VF)
**VF = C₃ ⊕ C_out**

`C₃` is the carry generated *into* bit position 3 (i.e., the carry out of the lower 3 bits' addition). `C_out` is the carry out of bit position 3. If these two carries differ, signed overflow has occurred — a positive result from two negative operands or vice versa. This is the textbook signed overflow condition, implemented literally in hardware as a single XOR gate.

The reason for **two adders**: the 7483 only exposes `C_out`. `C₃` is internal. Splitting the 4-bit operands as (X₀,X₁,X₂) into adder #1 and (X₃) into adder #2 — with adder #1's `C_out` wired to adder #2's `C_in` — makes `C₃` accessible as adder #1's output carry (`S₁` pin).

### Sign Flag (SF)
**SF = S₀** (MSB of the 4-bit output from adder #2). Directly reflects the most significant bit of the result, indicating whether it is negative in two's complement representation.

### Zero Flag (ZF)
**ZF = NOR(S₀ ∨ S₁, S₂ ∨ S₃)**

Computed by first ORing the output bit pairs `(S₀ ∨ S₁)` and `(S₂ ∨ S₃)`, then feeding both results into a NOR gate. The final output is 1 if and only if all four result bits are 0.

Flag behaviour for logical operations (AND, XOR, Complement A):
- **CF = 0**, **VF = 0** (cleared, as per assembly convention)
- **SF** and **ZF** set according to the result

---

## Block Diagram

```
                           cs2  cs1  cs0
                            │    │    │
              ┌─────────────▼────▼────▼──────────────────┐
              │           CONTROL LOGIC                   │
              │  S₁=cs2·cs0+cs1·cs0̄   S₂=cs2   S₃=cs0  │
              │  en₃=cs2+cs1·cs0̄   C_in=cs2̄·cs0+cs2·cs1·cs0̄│
              └──┬──────┬──────┬──────┬──────────────────┘
                 S₂     S₁     S₃    en₃     C_in
                 │      │      │      │         │
 A ─────┬──────► │      │      │      │         │
        │ A∧B ──►│      │      │      │         │
 B ─┬───┤ A⊕B ──►│      │      │      │         │
    │   │ Ā   ──►│      │      │      │         │
    │   │        │      │      │      │         │
    │   │  ┌─────▼──────▼──┐   │      │         │
    │   │  │ Dual 4×1 MUX  │   │      │         │
    │   │  │   (74153×2)   │   │      │         │
    │   │  └───────┬───────┘   │      │         │
    │   │          │ X[3:0]    │      │         │
    │   │    ┌─────▼──────────►│◄─────┘         │
    │   │    │  2×1 MUX       │                 │
    └───┴───►│  (74157)       │ Y[3:0]          │
    B / B̄   └─────────────────┘                 │
                     │                          │
         ┌───────────▼────────────┐             │
         │  4-bit Adder #1 (7483) │◄────────────┘
         │  X₀+Y₀, X₁+Y₁, X₂+Y₂ │  C_in
         │  → S₁,S₂,S₃  → C₃    │
         └───────────┬────────────┘
                     │ C₃ (carry into bit 3)
         ┌───────────▼────────────┐
         │  4-bit Adder #2 (7483) │
         │  X₃+Y₃+C₃            │
         │  → S₀(MSB) → C_out   │
         └──────┬──────┬─────────┘
                │      │
         ┌──────▼──────▼──────────────────────┐
         │           FLAG LOGIC               │
         │  CF = C_out                        │
         │  VF = C₃ ⊕ C_out      (7486 XOR)  │
         │  SF = S₀               (MSB)       │
         │  ZF = NOR(S₀∨S₁, S₂∨S₃) (7402)   │
         └────────────────────────────────────┘
                 Output: S₃ S₂ S₁ S₀, CF, VF, SF, ZF
```

---

## ICs Used

The final design uses **12 ICs total** — well within the target budget and achieved through deliberate K-map optimization and gate reuse:

| IC | Function | Qty | Role in Circuit |
|----|----------|:---:|-----------------|
| **7402** | Quad 2-input NOR | 1 | ZF computation (NOR of OR pairs) + NOT replacement |
| **7404** | Hex Inverter (NOT) | 1 | Generating B̄ (for Sub) and control signal inversions |
| **7408** | Quad 2-input AND | 2 | A∧B pre-computation + S₁ logic (cs2·cs0, cs1·cs0̄) |
| **7432** | Quad 2-input OR | 1 | en₃ logic (cs2 + cs1·cs0̄) + ZF intermediate ORs |
| **7483** | 4-bit Full Adder | 2 | Core arithmetic — split to expose C₃ for VF |
| **7486** | Quad 2-input XOR | 2 | A⊕B pre-computation + VF = C₃⊕C_out |
| **74153** | Dual 4×1 MUX | 2 | Selecting X_i from {A, A∧B, A⊕B, Ā} |
| **74157** | Quad 2×1 MUX | 1 | Selecting Y_i from {B, B̄, 0} |
| | **Total** | **12** | |

---

## Optimization — How We Got to 12 ICs

Minimizing IC count was an explicit requirement and a genuine engineering puzzle. Here is the exact optimization that shaved us from 13 to 12:

**The problem**: the Zero Flag computation required both an OR gate and a NOT gate — which would have meant using the 7432 (for OR) and keeping the 7404 (for NOT on top of OR), but the 7432 was only being used for 2 of its 4 available OR gates. Adding a 7404 just for one NOT felt wasteful.

**The solution**: replace the 7432 + 7404 pair for ZF with a single **7402 (NOR)**. A NOR gate is a natural OR-then-NOT in one operation. Two NOR gates connected to implement the OR intermediate stages, and the final NOR gate on top — all from one IC. As a bonus, the 7402 has four gates total, and the remaining gate handled the NOT operation that was going to require the 7404 for something else. The 7432 and the need for an extra 7404 both evaporated.

Additional savings came from:
- The 7486 (XOR) IC has 4 gates; using one gate for `A₁⊕B₁` in the X-path and another gate with `A₁` tied to both inputs (XOR with itself = 0, used as a tie-to-zero trick) to derive a needed 0
- Cross-checking K-map negations to identify shared terms between S₁, C_in, and en₃ that could share the same AND gate output

---

## Simulation

The full circuit was designed and verified in **Logisim 2.7.1** at IC level — meaning each component in the simulation corresponds to exactly one physical IC.

**Simulation files:** [`A2_Group2/A2_GROUP-2.circ`](A2_Group2/A2_GROUP-2.circ)
**Custom IC library:** [`A2_Group2/7400-lib.circ`](A2_Group2/7400-lib.circ)

### Running the Simulation

1. Install **Logisim 2.7.1** (or compatible version)
2. Open `A2_Group2/A2_GROUP-2.circ` in Logisim
3. If prompted for a missing library, load `7400-lib.circ` via **Project → Load Library → Logisim Library**
4. Go to **Simulate → Simulation Enabled** (`Ctrl+E`)
5. Set the three control inputs cs2, cs1, cs0 using the input pins
6. Set the 4-bit A and B inputs
7. Observe the 4-bit output and the four flag outputs (CF, VF, SF, ZF)

**Test cases to try:**

| A | B | cs2 cs1 cs0 | Operation | Expected Output | Expected Flags |
|---|---|-------------|-----------|-----------------|----------------|
| `0101` (5) | `0011` (3) | `000` | Add | `1000` (8) | CF=0, VF=0, SF=1, ZF=0 |
| `0101` (5) | `0011` (3) | `001` | Sub | `0010` (2) | CF=1, VF=0, SF=0, ZF=0 |
| `1010` | `1100` | `010` | AND | `1000` | CF=0, VF=0 |
| `1010` | `1100` | `100` | XOR | `0110` | CF=0, VF=0 |
| `0101` | — | `101` | Complement A | `1010` | ZF=0 |
| `0001` (1) | — | `110` | NEG A | `1111` (−1) | |
| `0000` | `0000` | `000` | Add | `0000` | ZF=1 |
| `0111` (7) | `0001` (1) | `000` | Add | `1000` (−8!) | VF=1 (signed overflow) |

---

## Hardware Implementation

The simulation was translated directly into a breadboard circuit using real TTL ICs, LEDs as output indicators, and DIP switches as inputs.

**Hardware inventory per the physical build:**
- 12 TTL ICs (as listed in the IC table above)
- 8 DIP switches (4 for A, 4 for B input)
- 3 toggle switches (cs0, cs1, cs2 control)
- 8 output LEDs (4 result bits + 4 flags)
- Breadboards, power rails (5V TTL), resistors (current limiting for LEDs)

**Build process:**
1. Power and ground rails established and verified with multimeter before any IC insertion
2. ICs placed with careful pin-number tracking (IC orientation, notch position)
3. Each IC's VCC (pin 14/16) and GND (pin 7/8) wired and voltage-checked before connections
4. Wiring done module by module — X_i/Y_i circuit first, then adder chain, then flags
5. Each stage verified with known inputs before proceeding to the next
6. Full end-to-end test with the complete truth table

---

## Challenges & The Moment It All Worked

Hardware implementation is where theory meets reality — and reality pushes back hard.

**Design iteration.** We actually built the hardware *twice*. Our first physical circuit had an unresolvable bug in one bit of the Y_i path — the signal wasn't behaving as expected despite the wiring looking correct. After exhausting debugging options, we made the call to tear it down and rebuild from scratch with a cleaner layout. That decision cost us an evening but paid off in clarity.

**The stubborn NOT gate.** On the second build, we hit a bug in the X₁ path: the NOT gate responsible for producing Ā₁ (terminals 5-6 of the 7404) was misbehaving — its output was floating high even when A₁ was driven high, which should have driven the output low. Swapping the IC didn't fix it. It was late at night and getting a replacement was not an option. Then one of us remembered that a XOR gate with one input tied high acts as an inverter (`A ⊕ 1 = Ā`). We had unused XOR gate terminals available on the second 7483 adder's S₃ output. We repurposed it as an inverter and got the correct Ā₁. The fix was not only functional — it was elegant.

**Switches and LEDs.** The DIP switches we initially used didn't seat properly in the breadboard; their pin spacing was slightly off-pitch, causing intermittent connections. We modified the pins and eventually switched to a different form factor. Several LEDs burned out early because we underestimated their forward current requirements and hadn't sized the series resistors correctly. Every burned-out LED was a lesson in datasheet reading.

**Being first.** We were among the first groups in our section to attempt the hardware demonstration. There was no one to ask "what worked for you." Every problem was ours to figure out from first principles — datasheet in hand, multimeter probing one node at a time.

**The moment it worked.** After hours of debugging across two builds, we set cs2=0, cs1=0, cs0=0 (Add), entered A=0101 and B=0011 on the switches, and watched the LEDs light up: `1000` — exactly 8. We cycled through every operation in the truth table. Every single one correct. The sense of accomplishment was genuinely surreal. We had taken a K-map on paper, converted it to Boolean expressions, placed ICs on a breadboard, soldered nothing but pressed everything into place with wires — and built something that *computed*. That feeling is why we study Computer Architecture.

---

## Team

**Section A2 · Group 02 · CSE 210 · BUET · October 2024**

| Student ID | Name | Contributions |
|------------|------|--------------|
| 2105043 | Monjur Hossain Khan | Hardware power/ground, pin tracking, wire management, verification, debugging |
| 2105046 | Mohammad Raihan Rashid | Design, optimization (IC reduction strategy), wire placement, hardware input, verification, report documentation |
| 2105047 | Himadri Gobinda Biswas | Design, Logisim simulation, pin tracking, verification, debugging, complete circuit diagram |
| 2105049 | Amit Saha | Design, Logisim simulation, hardware power/ground, wire management, wire placement, hardware input, debugging |
| 2105055 | Md. Abrar Jahin | Design verification, hardware power/ground, wire placement, hardware input, debugging, block diagram |

---

## Repository Structure

```
Assignment 1 ALU/
│
├── readme.md                              ← You are here
│
├── CSE210_Assignment1_v1.pdf             ← Original assignment specification
│
├── A2_Group2/
│   ├── A2_GROUP-2.circ                   ← Logisim simulation (open this)
│   └── 7400-lib.circ                     ← Custom 74xx IC library for Logisim
│
└── ALU_final_group2_Report/
    ├── main.tex                          ← LaTeX report source
    ├── main.pdf                          ← Compiled report
    └── Images/
        ├── xiyi.png                      ← X_i / Y_i circuit diagram
        ├── ModALU.png                    ← Modified two-adder system diagram
        ├── ALU.png                       ← Full ALU circuit diagram
        └── Combined.png                  ← Complete design overview
```

---

*Built from K-maps and logic gates. Debugged with a multimeter and sheer stubbornness.*
