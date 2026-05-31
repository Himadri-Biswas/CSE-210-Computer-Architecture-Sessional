# 32-Bit Floating Point Adder — Logisim Simulation

> **CSE 210 · Computer Architecture Sessional** | Section A2 · Group 01
> BUET · November 2024

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Number Format — Custom 32-bit IEEE 754](#number-format--custom-32-bit-ieee-754)
3. [Special Values Encoding](#special-values-encoding)
4. [Algorithm — Step by Step](#algorithm--step-by-step)
5. [Architecture Overview](#architecture-overview)
6. [Module Deep Dive](#module-deep-dive)
   - [Splitter](#1-splitter)
   - [Input Validator](#2-input-validator--pre-processor)
   - [Comparator](#3-comparator)
   - [Shifter](#4-shifter)
   - [Adder-Subtractor](#5-adder-subtractor-32-bit)
   - [Normalizer](#6-normalizer)
   - [Rounder](#7-rounder)
   - [Second Normalizer](#8-second-normalizer-post-rounding)
   - [Output Validator](#9-output-validator)
   - [Combiner](#10-combiner)
7. [Custom Library Components](#custom-library-components)
8. [ICs Used](#ics-used)
9. [How to Run in Logisim](#how-to-run-in-logisim)
10. [Test Cases](#test-cases)
11. [Challenges & Design Decisions](#challenges--design-decisions)
12. [Team](#team)
13. [Repository Structure](#repository-structure)

---

## Project Overview

This project is a **fully functional 32-bit Floating Point Adder (FPA)** designed and simulated in Logisim 2.7.1. The circuit takes two 32-bit floating-point numbers as inputs, conforming to a custom IEEE 754-like format, and produces their sum as a 32-bit output — handling every nuance the standard demands: exponent alignment, mantissa addition/subtraction, normalization, rounding, and full detection of Overflow, Underflow, NaN, Infinity, and Denormalized numbers.

What makes this project genuinely challenging is that floating-point arithmetic is not just binary addition. The two operands may have completely different exponents, different signs, and the result may require shifting by tens of bit positions in either direction before it is representable. Every one of those cases must be handled by dedicated combinational hardware wired together into a pipeline of modules — and every bug in any module propagates invisibly forward until the final output is wrong.

The design was built **entirely from scratch** in a modular style. Custom libraries for multiplexers, priority encoders, and comparators were built first, then composed into the functional modules (Normalizer, Rounder, Adder-Subtractor, etc.), and finally integrated into the complete FPA. Only the ALU sub-component was permitted to be taken from an external source, as specified in the assignment.

**Key highlights:**
- Custom 32-bit format: 1-bit sign, 9-bit exponent (bias=255), 22-bit mantissa
- Full pipeline: Validate → Compare → Align → Add/Sub → Normalize → Round → Re-normalize → Output
- Rounding implemented using Guard, Round, Sticky, and LSB bits
- Overflow, Underflow, NaN, Infinity, and Denormalized detection flags
- **186 ICs** in simulation (123 of which are 74157 quad 2×1 MUXes — unavoidable at 32-bit widths)
- 10 functional sub-modules, each designed and tested independently
- Simulator: **Logisim 2.7.1**

---

## Number Format — Custom 32-bit IEEE 754

The floating-point format used is a custom variant of IEEE 754 with **9-bit exponent** and **22-bit mantissa** (versus standard single-precision which uses 8-bit exponent and 23-bit mantissa):

```
 Bit 31     Bits 30–22        Bits 21–0
┌────────┬───────────────┬──────────────────────────┐
│  Sign  │   Exponent    │    Fraction / Mantissa   │
│  1 bit │    9 bits     │         22 bits          │
└────────┴───────────────┴──────────────────────────┘
```

The actual value of a normalized number is:

```
Value = (−1)^sign × (1 + fraction) × 2^(exponent − bias)
```

where **bias = 2^(9−1) − 1 = 255**.

The leading `1` in `(1 + fraction)` is the **hidden bit** — it is implicit and not stored in the 22 fraction bits. In our circuit, we explicitly prepend `01` to the fraction during computation to materialize the hidden bit, then strip it from the output using a left shifter.

**Representable range:**
| | Value | Binary |
|-|-------|--------|
| Largest positive normalized | ≈ 6.432 × 10^76 | `1.1111...1 × 2^255` |
| Smallest positive normalized | ≈ 3.454 × 10^−77 | `1.0000...0 × 2^−254` |
| Smallest denormalized | ≈ 1.0 × 10^−276 | `0.0000...1 × 2^−254` |

---

## Special Values Encoding

| Exponent (9-bit) | Fraction (22-bit) | Value Represented |
|:----------------:|:-----------------:|:-----------------:|
| `000000000` (0) | `0000...0` | ± Zero |
| `000000000` (0) | Non-zero | ± Denormalized number |
| `000000001`–`111111110` (1–510) | Anything | ± Normal floating-point number |
| `111111111` (511) | `0000...0` | ± Infinity |
| `111111111` (511) | Non-zero | NaN (Not a Number) |

---

## Algorithm — Step by Step

The complete floating-point addition algorithm runs through these stages:

```
┌──────────────────────────────────────────────────────────┐
│  Input A (32-bit)         Input B (32-bit)               │
└────────────┬──────────────────────┬──────────────────────┘
             ▼                      ▼
        ┌─────────┐           ┌─────────┐
        │Splitter │           │Splitter │
        └────┬────┘           └────┬────┘
             │ Sign, Exp, Frac     │ Sign, Exp, Frac
             └──────────┬──────────┘
                        ▼
               ┌─────────────────┐
               │ Input Validator │  → NaN / Inf / Denormalized flags
               └────────┬────────┘
                        │ (if valid)
                        ▼
               ┌─────────────────┐
               │   Comparator    │  → Larger, Smaller, Difference
               └────────┬────────┘
                        │
                        ▼
                ┌───────────────┐
                │    Shifter    │  → Right-shifts smaller significand
                └───────┬───────┘     by Difference bits
                        │
                        ▼
            ┌───────────────────────┐
            │   Adder-Subtractor    │  → Add (same sign) or Sub (diff sign)
            └───────────┬───────────┘     + Sign_of_Result
                        │
                        ▼
               ┌─────────────────┐
               │   Normalizer    │  → Shift until leading 1, adjust exp
               └────────┬────────┘     → Overflow / Underflow flags
                        │
                        ▼
                ┌───────────────┐
                │    Rounder    │  → Guard/Round/Sticky/LSB rounding
                └───────┬───────┘
                        │
                        ▼
            ┌────────────────────────┐
            │  Normalizer (2nd pass) │  → Re-normalize if rounding overflowed
            └────────────┬───────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │   Output Validator   │  → Final Overflow / Underflow
              └──────────┬───────────┘
                         │
                         ▼
                  ┌─────────────┐
                  │   Combiner  │  → Assemble 32-bit output
                  └─────────────┘
```

### The 7 Algorithmic Steps

**Step 1 — Exponent Alignment**
Compare the two exponents. The operand with the smaller exponent has its significand right-shifted by the difference. This ensures both mantissas are scaled to the same power of 2 before addition.

**Step 2 — Mantissa Add/Subtract**
If both operands have the **same sign**, add the significands. If they have **different signs**, subtract the smaller from the larger. The sign of the result is determined by which operand had the larger absolute magnitude.

**Step 3 — First Normalization**
The raw result of addition/subtraction may not be normalized (leading 1 not at bit 22). A priority encoder finds the position of the most significant set bit; the normalizer left-shifts until the `1` is in the correct position and adjusts the exponent accordingly. If a carry propagated out during addition, the result is right-shifted by 1 and the exponent incremented.

**Step 4 — Overflow / Underflow Check**
If the adjusted exponent exceeds 510 → **Overflow**. If it drops below 1 → **Underflow** (the result becomes denormalized or flushes to zero).

**Step 5 — Rounding**
The rounded-off bits that fell off the right side during shifting are captured as Guard (G), Round (R), and Sticky (S) bits. The LSB (L) of the kept mantissa is also considered. Rounding decision: **if G AND (R OR S OR L) → add 1 to the mantissa** (round-to-nearest-even). Only a single sticky bit was needed (verified through testing).

**Step 6 — Second Normalization**
Rounding can push a `1` into the carry position, making the mantissa `10.xxxxx...` — no longer normalized. A second normalizer pass handles this edge case by right-shifting and incrementing the exponent.

**Step 7 — Special Case Handling**
NaN and Infinity inputs bypass the normal computation path. Denormalized inputs are flagged at input validation time. The output combiner assembles the final sign, exponent, and fraction from the pipeline.

---

## Architecture Overview

```
                    32-bit Input A          32-bit Input B
                         │                       │
               ┌─────────▼──────┐    ┌───────────▼────────┐
               │   Splitter A   │    │     Splitter B     │
               │ SignA,ExpA,    │    │  SignB,ExpB,FracB  │
               │ FracA          │    └────────────────────┘
               └────────┬───────┘              │
                        │                      │
               ┌────────▼──────────────────────▼────┐
               │           Input Validator           │
               │  (NaN / Inf / Denorm detection)     │
               │  ValidFracA          ValidFracB      │
               └─────────────────┬───────────────────┘
                                 │
                   ┌─────────────▼──────────────┐
                   │         Comparator          │
                   │  9-bit magnitude compare    │
                   │  → Larger / Smaller / Diff  │
                   └──────────┬──────────────────┘
                              │
                    ┌─────────▼─────────┐
                    │      Shifter       │
                    │  Right-shift by   │
                    │  Diff positions   │
                    └─────────┬─────────┘
                              │
              ┌───────────────▼─────────────────┐
              │       Adder-Subtractor (32-bit)  │
              │  Add or Sub aligned mantissas    │
              │  based on sign bits              │
              └───────────────┬─────────────────┘
                              │ Result + SignOfResult
              ┌───────────────▼──────────────────┐
              │         Normalizer               │
              │  Priority Encoder → left shift   │
              │  + exponent adjustment           │
              │  → Overflow / Underflow          │
              └───────────────┬──────────────────┘
                              │ NormalizedFrac + NormalizedExp
              ┌───────────────▼──────────────────┐
              │            Rounder               │
              │  G, R, S, L bit rounding        │
              └───────────────┬──────────────────┘
                              │ RoundedFrac
              ┌───────────────▼──────────────────┐
              │     Normalizer (2nd pass)        │
              │  Handle post-round carry         │
              └───────────────┬──────────────────┘
                              │
              ┌───────────────▼──────────────────┐
              │        Output Validator          │
              │  Final overflow/underflow check  │
              └───────────────┬──────────────────┘
                              │
              ┌───────────────▼──────────────────┐
              │            Combiner              │
              │  Assemble Sign+Exp+Frac          │
              └──────────────────────────────────┘
                              │
                    32-bit Output + Flags
                  (Overflow, Underflow, NaN_Output,
                   INF_Output, Denormalized_Output)
```

---

## Module Deep Dive

### 1. Splitter

Two identical splitters (one per input) decompose each 32-bit input word into its three fields:
- `SignA` / `SignB` — bit 31
- `ExpA` / `ExpB` — bits 30:22 (9 bits)
- `FracA` / `FracB` — bits 21:0 (22 bits)

### 2. Input Validator / Pre-processor

This module classifies each input before any arithmetic is attempted. It consists of two sub-circuits:

**Single Input Checker** — examines one FP number:
- Uses a **PE_32×5 priority encoder** to check if the fraction contains any set bit (`Fraction_Contains_1`)
- Uses a **PE_10×4 priority encoder** on the exponent to check `Exponent_Contains_1` (any bit set) and `Exponent_All_1` (all bits set = 511)
- Uses a **12-bit ALU** for additional comparisons
- Produces `ValidFraction`: the fraction value with the hidden bit stripped / prepared for the pipeline

**Input Validator** — combines two single-input checkers and uses **7402 (NOR)** and **7408 (AND)** gates to generate six output flags for the pair:
- `Denormalized_Input_1` / `Denormalized_Input_2` — exponent all zeros, fraction non-zero
- `NaN_Input_1` / `NaN_Input_2` — exponent all ones, fraction non-zero
- `INF_Input_1` / `INF_Input_2` — exponent all ones, fraction all zeros

### 3. Comparator

A custom **comparator library** with two circuits:

**9-Bit Magnitude Comparator** (for exponents):
- Subtracts the two 9-bit exponents using a cascaded subtractor
- Outputs: `SMALLER` (9-bit value of smaller exponent), `LARGER` (9-bit value of larger exponent), `DIFFERENCE` (how many positions to shift), and a flag `X>Y` (which input had the larger exponent)
- Caps shift amount at 31 (`max shift 31`) — if the exponents differ by more than 22, the smaller operand completely vanishes into the shifted-off bits

**Smaller Fractions Magnitude Comparator** (for equal-exponent case):
- When both exponents are equal, the fractions must be compared to determine which operand is larger in absolute magnitude
- Outputs: `LargerFraction`, `SmallerFraction`, `CommonExponent`, `Difference`
- This is critical for correct sign determination when operands of opposite sign have the same exponent

### 4. Shifter

A **right-barrel-shifter** that shifts the smaller operand's significand rightward by exactly `Difference` bit positions. This aligns both mantissas to the same implicit exponent before addition.

The bits shifted off the right end are not discarded — they are passed to the **Rounder** as the Guard, Round, and Sticky bits.

### 5. Adder-Subtractor (32-bit)

A 32-bit signed add/subtract unit that:
- **Adds** the two aligned significands when they share the same sign
- **Subtracts** the smaller from the larger when signs differ

Built from:
- **IC 7404** (NOT) — to invert one operand for subtraction
- **IC 7408** (AND) — for control logic
- **IC 7432** (OR) — for control logic
- **MUX_25** (×2) — to select between add and subtract paths
- **IC 7408** (AND) — for carry-in generation

Outputs: a 32-bit `Result` and a single `Sign_of_Result` bit.

The sign of the result is derived by comparing the signs of the two inputs and which had the larger magnitude — this logic was one of the trickier bugs to fix (see Challenges section).

### 6. Normalizer

The most complex module. After addition/subtraction, the result mantissa may have its leading `1` anywhere from bit 0 to bit 31. The normalizer finds it and shifts it to the correct position (bit 22 for a 23-bit significand with hidden bit).

**Internal mechanism:**
1. A **32×5 priority encoder** identifies the position of the most significant set bit (MSB position encoded as a 5-bit number)
2. This tells the circuit whether to **left-shift** (MSB is below bit 22 — result got smaller) or **right-shift** (carry out of bit 22 — result got bigger)
3. A **7483 adder** adjusts the exponent by adding or subtracting the shift amount
4. MUX networks select the correct shifted version of the fraction

Overflow is set when the adjusted exponent exceeds 510 (all 1s). Underflow is set when it would go below 0. The `Denormalized` output flag is also produced here for near-zero results.

### 7. Rounder

Implements **round-to-nearest-even** using three rounding bits captured from the shifting process:
- **G (Guard)** — the first bit shifted off (MSB of discarded bits)
- **R (Round)** — the second bit shifted off
- **S (Sticky)** — OR of all remaining shifted-off bits (captures whether *any* non-zero bit was lost)
- **L (LSB)** — the least significant kept bit of the mantissa (used for the "round half to even" tie-breaking)

**Rounding decision:**
```
if (G AND (R OR S OR L)):
    mantissa = mantissa + 1   ← round up
else:
    mantissa unchanged         ← truncate
```

Built using **74157 (quad 2×1 MUX)** and **7402 (NOR)** gates. The addition of 1 to the mantissa is performed by a small ALU. Through careful analysis and testing, a single sticky bit was confirmed sufficient — no additional sticky bits were needed.

Output: `RoundedFrac` (22-bit)

### 8. Second Normalizer (post-rounding)

Rounding can cause the mantissa to overflow its 22 bits (e.g., `1.1111...1 + 1 = 10.0000...0`). When this happens, the result is right-shifted by 1 and the exponent incremented. This is a second, simpler normalization pass that only needs to handle the carry-out case. It also generates its own Overflow/Underflow flags if the incremented exponent exceeds 510.

### 9. Output Validator

Combines the Overflow and Underflow flags from both normalizer passes and produces the final status outputs:
- `Overflow` — exponent exceeded maximum
- `Underflow` — exponent went below minimum
- `NaN_Output`, `INF_Output`, `Denormalized_Output` — propagated from input flags or computed from result

### 10. Combiner

Assembles the final 32-bit output from:
- `Sign` (1 bit) — determined from the Adder-Subtractor's `Sign_of_Result`
- `NormalizedExp` (9 bits) — from the second normalizer
- `NormalizedFrac` (22 bits) — from the second normalizer

**Hidden bit trick**: During computation, `01` is prepended to the front of the 22-bit fraction to materialize the implicit leading `1` (creating a 24-bit working mantissa `01.xxxxxxxxxx`). On output, those two prepended bits are stripped using a **left shifter**, leaving the clean 22-bit fraction for the output word.

---

## Custom Library Components

Every sub-component below was built from scratch in Logisim (only the ALU was permitted as an external source):

### Multiplexer Library (`Multiplexer.circ`)
| Component | Purpose |
|-----------|---------|
| 9-bit 2×1 MUX | Selecting between 9-bit exponent values |
| 12-bit 2×1 MUX | Intermediate-width selection |
| 32-bit 2×1 MUX | Selecting between 32-bit significand paths |

### Comparator Library (`Comparator.circ`)
| Component | Purpose |
|-----------|---------|
| 9-bit Magnitude Comparator | Compares exponents, computes shift difference |
| Smaller Fractions Comparator | Compares mantissas when exponents are equal |

### Encoder Library (`Encoder.circ`)
Priority encoders find the position of the most significant set bit — the core of normalization:

| Component | Built From | Purpose |
|-----------|-----------|---------|
| 8×3 Priority Encoder | 74148 ICs | Base encoder |
| 16×4 Priority Encoder | Two cascaded 8×3 encoders | Mid-range encoding |
| 32×5 Priority Encoder | Two 16×4 + 74157 MUX select | Full 32-bit MSB detection |

### ALU (`ALU.circ` / `Manual ALU.circ`)
| Variant | Used In |
|---------|---------|
| 12-bit ALU | Input validation arithmetic |
| 16-bit ALU | Intermediate computations |
| 32-bit ALU | Main mantissa addition/subtraction |

All ALU instances are of identical construction (cascaded adder design), just different bit widths.

---

## ICs Used

The simulation uses a total of **186 IC-equivalent components**:

| IC / Component | Description | Qty |
|----------------|-------------|:---:|
| **IC 7402** | Quad 2-input NOR | 1 |
| **IC 7404** | Hex Inverter (NOT) | 2 |
| **IC 7408** | Quad 2-input AND | 6 |
| **IC 7432** | Quad 2-input OR | 6 |
| **IC 7486** | Quad 2-input XOR | 2 |
| **IC 74148** | 8-to-3 Priority Encoder | 20 |
| **IC 74157** | Quad 2×1 MUX | 123 |
| **ALU (12-bit)** | Cascaded adder | 11 |
| **ALU (16-bit)** | Cascaded adder | 4 |
| **ALU (32-bit)** | Cascaded adder | 3 |
| **Shifter (Left)** | Barrel shifter | 5 |
| **Shifter (Right)** | Barrel shifter | 3 |
| | **Total** | **186** |

> The 123 instances of IC 74157 are unavoidable — a 32-bit 2×1 MUX requires 8 quad-MUX ICs, and the design employs many such wide-bus MUXes throughout the pipeline. This count covers the entire component hierarchy, as only the ALU was outsourced — all other sub-components were built manually.

---

## How to Run in Logisim

### Prerequisites
- **Logisim 2.7.1** installed

### Steps

**1. Open the main circuit**

Open `A2_Group1/Floating_Point_Adder.circ` in Logisim.

**2. Load dependent libraries**

When prompted (or via **Project → Load Library → Logisim Library**), load the following circuit files from `A2_Group1/`:
- `7400-lib.circ` — TTL gate library
- `Multiplexer.circ` — custom MUX library
- `Comparator.circ` — comparator library
- `Encoder.circ` — priority encoder library
- `ALU.circ` — ALU library
- `Manual ALU.circ` — manual ALU
- `Normalizer.circ` — normalizer module
- `Rounder.circ` — rounder module
- `Utility.circ` — utility circuits
- `Mux.circ` — additional MUX circuits

**3. Enable simulation**

Go to **Simulate → Simulation Enabled** (`Ctrl+E`).

**4. Set inputs**

The circuit has two 32-bit input pins labeled **Input A** and **Input B**. Click each pin and enter the 32-bit binary representation of your floating-point number:

```
Bit 31    = Sign bit     (0 = positive, 1 = negative)
Bits 30-22 = Exponent   (9 bits, biased by 255)
Bits 21-0  = Fraction   (22-bit mantissa, hidden bit implicit)
```

**5. Read outputs**

- **32-bit Output** pin — the result in the same format
- **Overflow** flag — 1 if result exceeds max representable value
- **Underflow** flag — 1 if result is too small (below min normalized)
- **NaN_Output** — 1 if either input was NaN
- **INF_Output** — 1 if result is infinity
- **Denormalized_Output** — 1 if result is a denormalized number

**Converting a decimal to input format:**

```
Example: represent +5.75

1. 5.75 in binary = 101.11
2. Normalized: 1.0111 × 2^2
3. Sign = 0
4. Exponent = 2 + 255 (bias) = 257 = 100000001 in binary
5. Fraction = 01110000000000000000000 (22 bits, drop the leading 1)

32-bit input = 0 100000001 0111000000000000000000
             = 0100 0000 1011 1000 0000 0000 0000 0000
             = 0x40B80000
```

---

## Test Cases

| A (decimal) | B (decimal) | Expected Result | Notes |
|:-----------:|:-----------:|:---------------:|-------|
| +1.0 | +1.0 | +2.0 | Basic same-sign add |
| +1.5 | −0.5 | +1.0 | Different signs, A wins |
| −3.0 | +3.0 | 0.0 | Cancellation to zero |
| +0.1 | −0.1 | 0.0 | Near-cancellation (ZF) |
| Max positive | Max positive | Overflow | Exponent exceeds 510 |
| Min positive | −Min positive | 0 / Underflow | Near-underflow |
| +Infinity | +1.0 | +Infinity | Inf propagation |
| NaN | +1.0 | NaN | NaN propagation |
| +1.0 | −1.0 + ε | small positive | Catastrophic cancellation |
| +5.75 | +3.25 | +9.0 | Manual verification case |

---

## Challenges & Design Decisions

Building this FPA was one of the most intellectually demanding exercises we had undertaken. Theory is straightforward — hardware is not.

**Wrong sign with correct magnitude.** The most confusing bug: when input X and Y had different signs and |Y| > |X|, the output had the right magnitude but the wrong sign. After extended debugging, the root cause was that we were sending the raw fraction values to the comparator without first checking which sign each fraction belonged to. The FPA was therefore always assuming X > Y regardless of actual magnitude. The fix was routing both the fraction values *and* their sign bits through a pair of 2×1 MUX components before the comparator, so the larger signed value was correctly identified every time.

**Normalizer edge cases.** The normalizer was the most time-consuming module. Several edge cases caused silent failures:
- The case where both exponent and fraction are zero must **not** be classified as Denormalized (it's plain zero) — this required an explicit check.
- We initially planned to keep Denormalized and Infinity flags inside the normalizer for propagation, but reverted after testing revealed it made the control logic brittle. These flags are handled entirely at the input validator and output validator stages instead.

**Rounding and the second normalizer.** Rounding seems simple until you realize that adding 1 to a mantissa of the form `1.1111...1` produces `10.0000...0` — which shifts the exponent. The second normalization pass exists entirely for this edge case. Without it, a small class of inputs produces results that are off by exactly one ULP (unit in the last place) in the worst direction.

**The hidden bit materialization.** The IEEE 754 hidden bit is never stored — only implied. During computation, we prepend `01` to the front of the 22-bit fraction, making the working mantissa 24 bits wide with `01.xxxxxx` structure. On output, those two leading bits are stripped by a left shifter. Getting this accounting right across all shifts, additions, and normalizations was fiddly and required careful wire tracing.

**74157 MUX explosion.** Every wide-bus MUX (32-bit, selecting between two 32-bit buses) consumes 8 IC 74157 quad-MUX chips. With a pipeline that passes 32-bit values through many selection points, the MUX count ballooned to 123. This is not a design inefficiency — it is the unavoidable cost of implementing wide multiplexing at IC level.

**G, R, S bit theory vs practice.** We thoroughly learned and applied the Guard-Round-Sticky rounding theory. Through systematic testing, we confirmed that a single sticky bit (OR of all trailing discarded bits) is sufficient for our design. Using additional sticky bits would increase buffer capacity but did not change any test result, so we kept it minimal.

---

## Team

**Section A2 · Group 01 · CSE 210 · BUET · November 2024**

| Student ID | Name | Contributions |
|------------|------|--------------|
| 2105032 | Nawriz Ahmed Turjo | Rounder, Floating Point Adder integration, Utility circuits, IC optimization, Flags logic |
| 2105047 | Himadri Gobinda Biswas | Normalizer, Comparator, MUX library, Debugging & Testing, Floating Point Adder integration |
| 2105048 | Shams Hossain Simanto | Manual ALU (adder-subtractor), Priority Encoder library, Normalizer, Debugging, Wire labelling, Floating Point Adder integration |

---

## Repository Structure

```
Assignment 2 FPA/
│
├── readme.md                          ← You are here
│
├── 210_FP_Adder (1).pdf              ← Original assignment specification
│
└── A2_Group1/
    ├── Floating_Point_Adder.circ     ← Main FPA circuit (open this)
    │
    ├── Normalizer.circ               ← Normalizer module
    ├── Rounder.circ                  ← Rounder module
    │
    ├── ALU.circ                      ← 12/16/32-bit ALU library
    ├── Manual ALU.circ               ← 32-bit Adder-Subtractor
    ├── Comparator.circ               ← 9-bit & fraction comparators
    ├── Encoder.circ                  ← 8×3, 16×4, 32×5 priority encoders
    ├── Multiplexer.circ              ← 9/12/32-bit 2×1 MUX library
    ├── Mux.circ                      ← Additional MUX circuits
    ├── Utility.circ                  ← Miscellaneous utility circuits
    │
    ├── 7400-lib.circ                 ← 74xx TTL gate library
    │
    └── FPA_Report.pdf                ← Full technical report
```

---

*Designed bit by bit. Debugged wire by wire. Every edge case handled.*
