# ⚡ 4-Bit Carry Lookahead Adder (CLA) — Cadence Virtuoso | GPDK 180nm

<p align="center">
  <img src="https://img.shields.io/badge/Tool-Cadence%20Virtuoso-blue?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/PDK-GPDK%20180nm-green?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Type-Digital%20VLSI-orange?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Verified-DRC%20%7C%20LVS-brightgreen?style=for-the-badge"/>
</p>

---

## 📌 Table of Contents

- [Project Overview](#-project-overview)
- [CLA Adder — Theory & Working](#-cla-adder--theory--working)
- [Block Hierarchy](#-block-hierarchy)
- [Sub-Circuit 1: PG Logic (Propagate & Generate)](#-sub-circuit-1-pg-logic-propagate--generate)
- [Sub-Circuit 2: 3-Input XOR (Sum Logic)](#-sub-circuit-2-3-input-xor-sum-logic)
- [Sub-Circuit 3: Carry Logic (C_logic)](#-sub-circuit-3-carry-logic-c_logic)
- [Top-Level CLA Adder Schematic](#-top-level-cla-adder-schematic)
- [Layout Views](#-layout-views)
- [Tools & Technology](#-tools--technology)
- [Directory Structure](#-directory-structure)

---

## 📖 Project Overview

This project implements a **4-bit Carry Lookahead Adder (CLA)** at the transistor level using **Cadence Virtuoso** on the **GPDK 180nm** process node. The CLA overcomes the primary bottleneck of the Ripple Carry Adder (RCA) — carry propagation delay — by computing all carry signals in parallel using dedicated combinational logic.

The design is hierarchical, composed of three custom sub-cells:

| Sub-Cell | Function | Inputs | Outputs |
|---|---|---|---|
| `PG` | Propagate & Generate logic | A, B | P = A⊕B, G = A·B |
| `XOR` (3-input) | Sum computation | A, B, Cin | S = A⊕B⊕Cin |
| `C_logic` | Carry computation | A, B, Cin | Cout = G + P·Cin |

All cells were schematic-designed, symbolized, instantiated in the top-level CLA schematic, and then laid out with full **DRC** and **LVS** verification.

---

## 🧮 CLA Adder — Theory & Working

### Why CLA Over Ripple Carry?

In a standard Ripple Carry Adder, each full adder must wait for the carry from the previous stage:

```
C1 = f(A0, B0, Cin)
C2 = f(A1, B1, C1)     ← must wait for C1
C3 = f(A2, B2, C2)     ← must wait for C2
...
```

This creates a **carry chain delay** of O(n) gate delays for an n-bit adder.

### CLA Principle

CLA introduces two intermediate signals per bit:

- **Propagate (P):** `Pᵢ = Aᵢ ⊕ Bᵢ` — the carry *propagates* through bit i if one (but not both) inputs are 1.
- **Generate (G):** `Gᵢ = Aᵢ · Bᵢ` — the carry is *generated* at bit i if both inputs are 1.

Using P and G, all carry bits are computed in **parallel** (not sequentially):

```
C0 = Cin
C1 = G0 + P0·Cin
C2 = G1 + P1·G0 + P1·P0·Cin
C3 = G2 + P2·G1 + P2·P1·G0 + P2·P1·P0·Cin
C4 = G3 + P3·G2 + P3·P2·G1 + P3·P2·P1·G0 + P3·P2·P1·P0·Cin
```

The final **Sum** bits are:

```
Sᵢ = Aᵢ ⊕ Bᵢ ⊕ Cᵢ  =  Pᵢ ⊕ Cᵢ
```

### Performance Advantage

| Adder Type | Carry Delay (n-bit) | Sum Delay |
|---|---|---|
| Ripple Carry | O(n) | O(n) |
| Carry Lookahead | O(1) — constant | O(1) |

For a 4-bit adder, the CLA reduces the critical path from **4 full-adder stages** to effectively **2 logic levels** for carry computation.

---

## 🏗 Block Hierarchy

```
CLA_Adder (Top)
├── PG × 4          → Computes Pᵢ and Gᵢ for each bit
├── XOR (3-input) × 4  → Computes final Sum Sᵢ = Pᵢ ⊕ Cᵢ
└── C_logic × 4     → Computes Carry Cᵢ₊₁ = Gᵢ + Pᵢ·Cᵢ
```

---

## 🔷 Sub-Circuit 1: PG Logic (Propagate & Generate)

### Schematic

![PG Logic Schematic](PG_logic.png)

### Working

The PG cell takes two single-bit inputs **A** and **B** and produces:

- **P (Propagate):** `P = A ⊕ B` — implemented using an XOR gate
- **G (Generate):** `G = A · B` — implemented using an AND gate

Both P and G signals are computed simultaneously and forwarded to both the carry logic and the XOR sum logic.

| A | B | P = A⊕B | G = A·B |
|---|---|---------|---------|
| 0 | 0 | 0 | 0 |
| 0 | 1 | 1 | 0 |
| 1 | 0 | 1 | 0 |
| 1 | 1 | 0 | 1 |

> **Note:** P = 1 and G = 0 means carry *passes through*. G = 1 always means a new carry is *generated* regardless of Cin.

### Layout

![PG Logic Layout](PG_opt.png)

The layout uses standard CMOS XOR and AND cell implementations in GPDK 180nm. Power rails (VDD/VSS) run horizontally at the top and bottom, with signal routing in Metal 1/Metal 2.

---

## 🔷 Sub-Circuit 2: 3-Input XOR (Sum Logic)

### Schematic

![3-Input XOR Schematic](3_IN_xor.png)

### Working

The 3-input XOR computes the **sum bit** for each stage:

```
S = A ⊕ B ⊕ C
```

This is implemented as **two cascaded 2-input XOR gates**:

- **Stage 1:** `out1 = A ⊕ B`
- **Stage 2:** `S = out1 ⊕ C`

In the context of the CLA adder, the inputs are:
- **A** = Aᵢ (bit from operand A)
- **B** = Bᵢ (bit from operand B)
- **C** = Cᵢ (carry-in from lookahead carry logic)

Both XOR stages share the same **VDD** and **VSS** rails, connected at the top and bottom respectively.

| A | B | C | S = A⊕B⊕C |
|---|---|---|-----------|
| 0 | 0 | 0 | 0 |
| 0 | 0 | 1 | 1 |
| 0 | 1 | 0 | 1 |
| 0 | 1 | 1 | 0 |
| 1 | 0 | 0 | 1 |
| 1 | 0 | 1 | 0 |
| 1 | 1 | 0 | 0 |
| 1 | 1 | 1 | 1 |

### Layout

![3-Input XOR Layout](3_in_xor_opt_layout.png)

The layout shows two XOR cell instances placed side by side, with shared power straps at the top. Signal interconnects between Stage 1 and Stage 2 are routed in Metal 1.

---

## 🔷 Sub-Circuit 3: Carry Logic (C_logic)

### Schematic

![Carry Logic Schematic](CARRY_logic.png)

### Working

The `C_logic` cell computes the **carry output** for each bit position using the P and G signals from the PG cell:

```
Cout = G + P · Cin
```

This is realized with:
- An **AND gate** → computes `P · Cin`
- An **OR gate** → computes `G + (P · Cin)` → final carry output

The PG cell provides P and G; Cin is the lookahead carry from the previous stage.

| G | P | Cin | Cout = G + P·Cin |
|---|---|-----|-----------------|
| 0 | 0 | 0 | 0 |
| 0 | 0 | 1 | 0 |
| 0 | 1 | 0 | 0 |
| 0 | 1 | 1 | 1 |
| 1 | 0 | 0 | 1 |
| 1 | 0 | 1 | 1 |
| 1 | 1 | 0 | 1 |
| 1 | 1 | 1 | 1 |

### Layout

![Carry Logic Layout](carry_opt.png)

The AND and OR gate instances are placed compactly, with P, G, and Cin signal lines routed in Metal 1. The output carry `C` is tapped to the next stage's `Cin` in the top-level CLA netlist.

---

## 🔷 Top-Level CLA Adder Schematic

### Schematic

![CLA Adder Schematic](CLA_adder_schematic.png)

### Interconnection

The top-level schematic instantiates 4 slices, one per bit (bit 0 → bit 3). Each slice contains:

1. A **3-input XOR** for the sum bit Sᵢ
2. A **C_logic** cell for the lookahead carry Cᵢ₊₁

**Signal flow for bit i:**

```
Inputs: Aᵢ, Bᵢ, Cᵢ (carry-in computed by lookahead)
         │
         ▼
     [PG Cell] ──→ Pᵢ, Gᵢ
         │
   ┌─────┴──────┐
   ▼            ▼
[3-in XOR]   [C_logic]
   │            │
   ▼            ▼
   Sᵢ (Sum)   Cᵢ₊₁ (Carry to next PG+C_logic stage)
```

Carry signals C0 through C3 are pre-computed in parallel — not chained through full adder stages — giving the CLA its speed advantage.

**Port list:**

| Port | Direction | Width | Description |
|---|---|---|---|
| A[3:0] | Input | 4-bit | Operand A |
| B[3:0] | Input | 4-bit | Operand B |
| Cin | Input | 1-bit | Carry-in (LSB) |
| S[3:0] | Output | 4-bit | Sum bits |
| Cout | Output | 1-bit | Final carry-out |

---

## 🗺 Layout Views

### CLA Adder Full Layout

![CLA Adder Layout](CLA_adder_layout.png)

The full CLA layout consists of four bit-slices arranged vertically. Each slice contains the PG, XOR, and C_logic cells placed in a standard-cell row style:

- **Top power rail:** VDD (Metal 1, horizontal)
- **Bottom power rail:** VSS (Metal 1, horizontal)
- **Signal routing:** Metal 1 (horizontal) and Metal 2 (vertical)
- **Cell abutment:** Adjacent cells share power rails to minimize area

**Design Rule Check (DRC):** Passed — No violations  
**Layout vs. Schematic (LVS):** Passed — Netlists match

---

## 🛠 Tools & Technology

| Item | Details |
|---|---|
| EDA Tool | Cadence Virtuoso 6.x |
| Simulator | Spectre |
| Process Node | GPDK 180nm CMOS |
| Supply Voltage | 1.8V |
| Logic Style | Static CMOS |
| Verification | DRC, LVS |

---

## 📁 Directory Structure

```
CLA_Adder_Cadence/
│
├── schematic/
│   ├── PG_logic.png            # PG cell schematic
│   ├── 3_IN_xor.png            # 3-input XOR schematic
│   ├── CARRY_logic.png         # Carry logic schematic
│   └── CLA_adder_schematic.png # Top-level CLA schematic
│
├── layout/
│   ├── PG_opt.png              # PG cell layout
│   ├── 3_in_xor_opt_layout.png # XOR cell layout
│   ├── carry_opt.png           # Carry logic layout
│   └── CLA_adder_layout.png    # Full CLA layout
│
└── README.md
```

---

## 🧑‍💻 Author

**Raghul**  
B.E. / B.Tech — VLSI Design  
Cadence Virtuoso | GPDK 180nm | Digital CMOS Design

---

> *Designed and verified as part of a digital VLSI course project using Cadence Virtuoso on the GPDK 180nm process node.*
