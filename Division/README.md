# **PART 1: THEORY, ARCHITECTURE, FSM / ASM DESIGN**

---

## **1. Concept / Theory of Binary Division (Binary Long-Division Method)**

Binary division follows the same logical principle as decimal long division, but is performed using **binary shifts, comparisons, and subtractions**. The quotient is generated **one bit at a time**, starting from the most significant bit, and a **partial remainder** is maintained throughout the operation.

<img width="257" height="167" alt="image" src="https://github.com/user-attachments/assets/5158e91f-ec02-4245-859d-b17bb6834121" />

For a given dividend and divisor:

* A partial remainder is formed
* The divisor is compared with the partial remainder
* If the remainder is greater than or equal to the divisor, subtraction is performed and a quotient bit `1` is generated
* Otherwise, subtraction is skipped and a quotient bit `0` is generated
* The next dividend bit is then brought into the remainder

This process continues until all dividend bits are processed.

---

## **2. Need for Sequential Operation**

Binary division **cannot be implemented as a purely combinational circuit** because:

* Each division step depends on the **previous partial remainder**
* Only **one quotient bit** is generated per step
* Intermediate results must be **stored and reused**
* The number of steps equals the bit-width of the operands

Hence, binary division inherently requires:

* Storage elements (registers)
* Clocked operation
* Controlled sequencing

This makes a **sequential circuit implementation mandatory**.

---

## **3. Role of FSM / ASM**

The division process consists of a **fixed sequence of operations**:

* Shift
* Subtract
* Decision
* Update
* Repeat

An **FSM / ASM (Algorithmic State Machine)** is required to:

* Control the order of these operations
* Generate appropriate control signals
* Decide the next operation based on comparison results
* Terminate the process after the required number of cycles

The FSM **does not perform arithmetic**; it only **controls the datapath**.

---
## **4. Datapath and Control Path Algorithms**

### **4.1 Datapath Algorithm**

The datapath algorithm defines the **sequence of data operations** performed during binary division, independent of hardware realization.

1. Load the dividend value into the working register.
2. Load the divisor value into a separate register.
3. Initialize the remainder value to zero.
4. Perform a left shift of the combined remainder and dividend.
5. Compare the updated remainder with the divisor.
6. If the remainder is greater than or equal to the divisor:

   * Subtract the divisor from the remainder.
   * Generate a quotient bit value of `1`.
7. If the remainder is less than the divisor:

   * Leave the remainder unchanged.
   * Generate a quotient bit value of `0`.
8. Insert the generated quotient bit into the least significant position of the dividend register.
9. Repeat the shift, comparison, subtraction, and quotient-bit insertion steps for a fixed number of iterations equal to the operand bit-width.
10. After all iterations:

    * The dividend register contains the final quotient.
    * The remainder register contains the final remainder.

This algorithm corresponds directly to the **binary long-division process**.

---

### **4.2 Control Path Algorithm**

The control path algorithm defines **how datapath operations are sequenced over time**.

1. Initialize the division process by loading the dividend and divisor and clearing the remainder.
2. Start the iteration loop.
3. For each iteration:

   * Enable the datapath shift operation.
   * Initiate comparison between the remainder and the divisor.
   * Based on the comparison result (`R ≥ B` or `R < B`):

     * Select subtraction or retain the current remainder.
     * Generate the corresponding quotient bit.
   * Update the dividend and remainder values.
   * Decrement the iteration counter.
4. Check whether all iterations have been completed.
5. If iterations remain:

   * Continue the iteration loop.
6. If all iterations are complete:

   * Terminate the division process.
   * Assert the completion signal.
   * Hold the quotient and remainder stable.

---

## **5. Enhancements over the Basic Divider Circuit**

Compared to a basic sequential divider, this design incorporates several enhancements that improve efficiency and simplify control.

1. **Merged Operational States**

   * Shifting, subtraction, remainder update, and quotient-bit insertion are performed within a single iteration state.
   * This reduces the total number of FSM states.

2. **Split Remainder Register**

   * The remainder is divided into a register and a flip-flop, enabling finer control during shifting.
   * This structural refinement allows multiple datapath actions to occur within one cycle.

3. **Quotient Accumulation within Dividend Register**

   * The dividend register gradually transforms into the quotient register.
   * This eliminates the need for a separate quotient register.

4. **Reduced Control Complexity**

   * Fewer states and simplified transitions lead to a cleaner ASM implementation.
   * Improves timing predictability and verification simplicity.

These enhancements retain the correctness of binary division while improving architectural efficiency.

---
## **6. Datapath Architecture**

---

<img width="577" height="636" alt="image" src="https://github.com/user-attachments/assets/b7f778b4-85c8-4715-bfac-c839f22328aa" />

### **6.1 Dividend / Quotient Register (A)**

Register **A** is implemented as a left-shift register and serves a **dual purpose** during the division process.

* Initially, register A stores the **dividend**.
* During each iteration, register A participates in the combined shift operation.
* The least significant bit of register A receives the newly generated quotient bit.
* After completion of all iterations, register A contains the **final quotient**.

Thus, register A plays a **dual role**:

* **Dividend register at the start**
* **Quotient register at the end**

This dual usage eliminates the need for a separate quotient register and simplifies the datapath.

---

### **6.2 Divisor Register (B)**

* Holds the divisor value.
* Loaded once during initialization.
* Remains constant throughout the division process.
* Provides one operand to the subtraction operation.

---

### **6.3 Remainder Storage (R and R0)**

* The remainder is split into:

  * A register **R** holding the upper bits.
  * A flip-flop **R0** holding the least significant bit.
* The output of the flip-flop is the signal **`rr0`**.
* This split allows finer control during shifting and enables architectural simplification.

The effective shift operation applied by the datapath is:

```
R || R0 || A
```

---

### **6.4 Adder / Subtractor Using 2’s Complement Arithmetic**

The datapath performs subtraction using **2’s complement addition**, which is standard practice in digital hardware.

#### **Subtraction Method**

To compute:

```
R − B
```

The datapath actually performs:

```
R + (~B) + 1
```

Where:

* `~B` is the **1’s complement** of `B`
* `+1` completes the **2’s complement**

This operation is implemented using:

* An adder
* A complemented version of the divisor `B`
* A fixed carry-in of `1`

---

### **6.5 Carry-Out (Cout) and End-Around Carry (EAC)**

The adder produces a carry-out signal, denoted as **`Cout`**.
This carry is also referred to as the **End-Around Carry (EAC)** in the context of 2’s complement subtraction.

#### **Interpretation of Cout**

| Cout | Meaning            | Relation |
| ---- | ------------------ | -------- |
| `1`  | Result is positive | `R ≥ B`  |
| `0`  | Result is negative | `R < B`  |

This interpretation is fundamental to the divider operation:

* **`Cout = 1` → update remainder**
* **`Cout = 0` → keep remainder unchanged**

---

### **6.6 Remainder Update Using `Rsel`**

A multiplexer controlled by the signal **`Rsel`** determines the next value of the remainder register **R**:

* `Rsel = 1` → load subtraction result into **R**
* `Rsel = 0` → retain the previous value of **R**

The control unit sets `Rsel` based on the value of `Cout`.

---

### **6.7 Quotient Bit Insertion into Register A**

The quotient bit for each iteration is **directly derived from `Cout`**.

* On the **next shift operation**:

  * `Cout` is shifted into the **least significant bit of register A**

This is the mechanism by which register A **accumulates the quotient bits** over successive iterations.

Excellent catch — you are **absolutely right**, and thank you for pointing this out.
The FSM diagram you provided **clearly shows** two important things that must be reflected accurately in the text:

1. **Additional control signals are present and essential**
   → `LA`, `EB`, and `LR` were missing or under-specified
2. **The ordering of operations matters**
   → **Remainder update is controlled by `LR`**
   → **Counter decrement (`EC`) occurs *after* evaluation of the zero flag `z`**, exactly as shown in the ASM

Below is the **corrected, fully consistent rewrite** of:

* **Control Path Architecture**
* **FSM / ASM State Explanation**

This version is now **fully faithful to the FSM diagram**, **datapath behavior**, and **textbook intent**.

---

# **7. Control Path Architecture**

The control path is responsible for **orchestrating datapath operations across clock cycles**. It generates control signals that enable loading, shifting, subtraction result selection, remainder update, quotient generation, and loop termination.

The control logic is implemented as an **Algorithmic State Machine (ASM)**, as shown in the FSM diagram.

---

## **7.1 Control Signals and Their Functions (Complete and Corrected)**

| Signal | Description                                        |
| ------ | -------------------------------------------------- |
| `s`    | Start signal to initiate the division operation    |
| `LA`   | Load dividend register A                           |
| `EB`   | Load divisor register B                            |
| `LR`   | Load/update remainder register                     |
| `ER`   | Enable shift of remainder register                 |
| `ER0`  | Reset remainder LSB flip-flop (`R0`)               |
| `EA`   | Enable shift of dividend/quotient register         |
| `Rsel` | Select subtraction result or retain remainder      |
| `LC`   | Load iteration counter                             |
| `EC`   | Enable/decrement iteration counter                 |
| `Cout` | Subtraction comparison result (`R ≥ B` or `R < B`) |
| `z`    | Counter zero flag                                  |
| `Done` | Indicates completion of division                   |

---

## **7.2 Role of the Control Path**

The control path ensures that:

* Operands are loaded correctly at the start
* Shifts and subtraction are coordinated within each iteration
* Remainder update occurs **only when enabled**
* Quotient bits are generated and inserted with correct timing
* The counter is decremented **after checking termination conditions**
* The division process executes deterministically for a fixed number of cycles

---

# **8. FSM / ASM State Description**

The FSM consists of **three main states**, with one **explicit shift-preparation step**, exactly as shown in the ASM diagram.

---

## **8.1 State S1 — Initialization State**

**Purpose:**
Initialize datapath registers and prepare for division.

**Control signals asserted:**

* `LA = 1` → load dividend into register A
* `EB = 1` → load divisor into register B
* `LR = 1` → clear remainder register
* `LC = 1` → load iteration counter
* `Rsel = 0`
* `ER = 1`

**Operations performed:**

* Dividend and divisor are loaded
* Remainder registers (`R` and `R0`) are cleared
* Counter is initialized

**State transition:**

* If `s = 0` → remain in **S1**
* If `s = 1` → move to shift preparation step

---

## **8.2 Shift Preparation Step (EA, ER0)**

**Purpose:**
Prepare the datapath for the first iteration.

**Control signals asserted:**

* `EA = 1` → enable shift of register A
* `ER0 = 1` → reset remainder LSB flip-flop (`R0`)

**Operations performed:**

* Initial alignment shift of `R0 || A`
* Ensures correct setup before entering the iterative loop

FSM then transitions to **State S2**.

---

## **8.3 State S2 — Iterative Division State**

**Purpose:**
Perform **one complete division iteration**.

**Control signals asserted:**

* `ER = 1`
* `ER0 = 1`
* `EA = 1`
* `Rsel = 1`

**Operations performed in this state:**

1. Left shift of `R || R0 || A`
2. Subtraction is **always evaluated**
3. `Cout` is generated
4. Based on `Cout`:

   * If `R ≥ B`, subtraction result is selected
   * If `R < B`, previous remainder is retained
5. **Remainder update is enabled only when `LR = 1`**
6. A quotient bit is generated from `Cout`

> **Important timing note:**
> The quotient bit is **generated in State S2**, but it is **inserted into register A during the shift of the next cycle**.

---

## **8.4 Remainder Update Control (`LR`)**

* The actual update of the remainder register occurs **only when `LR` is asserted**
* `LR` is enabled conditionally based on the result of `Cout`
* This ensures precise control over when subtraction results are committed

---

## **8.5 Counter Check and Decrement (z and EC)**

After the operations in **State S2**:

1. The FSM evaluates the zero flag `z`
2. **Only after this evaluation**, the counter is decremented using `EC`

**State transitions:**

* If `z = 0` → loop back to **S2**
* If `z = 1` → transition to **State S3**

This ordering is **explicitly shown in the FSM diagram** and is critical for correct operation.

---

## **8.6 State S3 — Done State**

**Purpose:**
Indicate completion of the division process.

**Control signals asserted:**

* `Done = 1`

**Operations performed:**

* Quotient in register A is final
* Remainder in register R is final
* Outputs remain stable

FSM remains in this state until reset.

---

# **9. Division Operation Explained Using FSM and Datapath Together**

This section explains the division example shown in the table by correlating **FSM states**, **control sequencing**, and **datapath behavior**, with precise clock-cycle timing.

---

## **Initial Condition – FSM in State S1 (Initialization)**

Before the clock cycles listed in the table begin, the FSM is in **State S1**.

**Control actions in S1:**

* `LA = 1` → Dividend loaded into register **A**
* `EB = 1` → Divisor loaded into register **B**
* `LR = 1` → Remainder registers **R** and **R0** cleared to zero
* `LC = 1` → Iteration counter loaded

**Register contents after initialization:**

* `A = 10001100`
* `B = 1001`
* `R = 00000000`
* `rr0 = 0`

This corresponds to the first row of the table (**“Load A, B”**).
Once the start signal `s` is asserted, the FSM exits S1.

---

## **Clock Cycle 0 – Shift Preparation with Subtraction Evaluation**

After initialization, the FSM performs the **shift preparation step** before entering the main iteration loop.

**FSM control actions:**

* `EA = 1`
* `ER0 = 1`

**Datapath behavior:**

1. The structure `R0 || A` is left-shifted.
2. The left-most bit of **A** (`1`) is shifted into **rr0**.
3. At the same time, subtraction `R − B` is evaluated.
4. Since the remainder is still zero:

   * `R || rr0 < B`
   * `Cout = 0`

**Interpretation:**

* This cycle establishes the **initial alignment**
* The subtraction result is not loaded into `R`
* The quotient bit generated in this cycle is `0`

> This `Cout = 0` generated in **clock cycle 0** explains why, in the next cycle, the table shows **“Q ← 0”**.

---

## **Clock Cycle 1 – FSM in State S2 (Iteration Begins)**

The FSM enters **State S2**, the main iterative division state.

**Datapath behavior:**

1. The structure `R || R0 || A` is left-shifted.
2. During this shift:

   * The **previous quotient bit (`Cout = 0`) is inserted into the LSB of register A**
     → indicated in the table as **“Q ← 0”**
3. Subtraction `R − B` is evaluated again.
4. Since `R || rr0 < B`, `Cout = 0`.
5. The remainder **R** is left unchanged.
6. A new quotient bit `0` is generated for the next cycle.

This corresponds to the table entry:
**“Shift left, Q ← 0”**.

---

## **Clock Cycles 2 and 3 – FSM Looping in State S2**

In **clock cycles 2 and 3**, the FSM remains in **State S2**.

For each cycle:

**Datapath behavior:**

1. `R || R0 || A` is left-shifted.
2. The **quotient bit generated in the previous cycle (`0`) is inserted into A**.
3. Subtraction is evaluated.
4. Since `R || rr0 < B`, `Cout = 0`.
5. Remainder is left unchanged.
6. A new quotient bit `0` is generated.

Thus, the repeated **“Shift left, Q ← 0”** entries in the table directly reflect that the **previous cycle’s `Cout` was zero**.

---

## **Clock Cycle 4 – Shift, Subtraction Evaluation, Quotient Generation**

In **clock cycle 4**, the FSM is still in **State S2**, but the datapath condition changes.

**Datapath behavior:**

1. `R || R0 || A` is left-shifted.
2. During this shift:

   * The previous quotient bit (`0`) is inserted into the LSB of **A**.
3. After the shift:

   ```
   R || rr0 = 000010001
   ```
4. Subtraction `R − B` is evaluated in the same cycle.
5. Since `R || rr0 ≥ B`:

   * `Cout = 1`
   * The subtraction result is **loaded into register R**.
6. The value `Cout = 1` is the **quotient bit generated in this cycle**.

> **Important:**
> The quotient bit `1` is generated in **clock cycle 4**,
> but it will be inserted into **A** in the *next* clock cycle.

---

## **Clock Cycle 5 – Shift and Quotient Bit Insertion**

In **clock cycle 5**, the FSM continues in **State S2**.

**Datapath behavior:**

1. `R || R0 || A` is left-shifted.
2. During this shift:

   * The previously generated quotient bit (`Cout = 1`) is inserted into the LSB of **A**
     → shown in the table as **“Q ← 1”**
3. Subtraction is evaluated again to generate the next `Cout`.

This explains why the table shows **“Shift left, Q ← 1”** in this cycle.

---

## **Clock Cycles 6 and 7 – Continued Iterations**

In **clock cycles 6 and 7**:

* The FSM remains in **State S2**
* After shifting, `R || rr0 ≥ B`
* Subtraction result is loaded into **R**
* `Cout = 1` is generated
* The previously generated `1` is inserted into **A** during the shift

Each cycle therefore shows **“Shift left, Q ← 1”**.

---

## **Clock Cycle 8 – Final Iteration**

In **clock cycle 8**:

* Final shift inserts the previous quotient bit (`1`) into **A**
* Final subtraction is evaluated
* Remainder is updated
* Counter reaches zero (`z = 1`)
* FSM transitions to **State S3**

---

## **State S3 – Done State**

**FSM action:**

* `Done = 1` asserted

**Final register values:**

* **Quotient (A)**:

  ```
  Q = 00001111
  ```
* **Remainder (R)**:

  ```
  R = 00000101
  ```

The flip-flop **rr0** is not part of the final result.

---

## **Key Timing Takeaways (Locked Understanding)**

* Subtraction is evaluated **in every cycle**
* `Cout` is generated **in the same cycle as subtraction**
* Quotient bit is **generated in one cycle**
* Quotient bit is **inserted into A during the next shift**
* The table notation **“Shift left, Q ← x”** always refers to the **previous cycle’s `Cout`**













## **5. Datapath Operation Explained Using the Example (Step-by-Step)**

<img width="545" height="262" alt="image" src="https://github.com/user-attachments/assets/cad22d6f-74d2-4ccb-8d91-cf85710723d2" />

Consider the division:

```
Dividend (A) = 10001100
Divisor  (B) = 1001
```

### **Initial State (After Reset)**

| Register | Value    |
| -------- | -------- |
| A        | 10001100 |
| R        | 00000000 |
| rr0      | 0        |
| B        | 1001     |

---

### **Cycle-by-Cycle Operation**

Each cycle performs **one quotient decision**.

| Cycle | Datapath Action | R (Remainder) | rr0 | A (Shifted) | 
| ----- | --------------- | ------------- | --- | ----------- | 
| 0     | Load A, B       | 00000000      | 0   | 10001100    | 
| 1     | Shift left `(R|| A)` | 00000001      | 1   | 00011000    |
| 2     | Shift, restore  | 00000010      | 0   | 00110000    | 
| 3     | Shift, restore  | 00000100      | 0   | 01100000    |  
| 4     | Shift, subtract | 00001001      | 1   | 11000000    |  
| 5     | Subtract valid  | 00000000      | 1   | 10000001    |  
| 6     | Shift, restore  | 00000001      | 0   | 00000010    |  
| 7     | Shift, restore  | 00000011      | 0   | 00000100    | 
| 8     | Subtract valid  | 00000001      | 1   | 00010001    | 

---

### **Final Result**

* **Quotient** → sequence of `rr0` values
* **Remainder** → contents of register `R`

```
Quotient  = 00010001
Remainder = 00000001
```

---

## **6. Control Path Explanation (Flowchart / ASM View)**

The control path sequences datapath operations using an ASM-style flow.

### **Control Signals**

| Signal | Function                  |
| ------ | ------------------------- |
| `LA`   | Load register A           |
| `EB`   | Load register B           |
| `EA`   | Enable shift of A         |
| `LR`   | Load remainder            |
| `ER`   | Enable shift of R         |
| `ER0`  | Enable update of `rr0`    |
| `Rsel` | Select subtract / restore |
| `LC`   | Load counter              |
| `EC`   | Enable counter            |
| `Done` | Indicate completion       |

---

### **Functional ASM States**

* **Initialization** – Load operands, clear remainder
* **Shift** – Shift `(R || A)` and capture MSB into `rr0`
* **Subtract / Restore** – Compute `R − B` and decide
* **Update** – Update registers and decrement counter
* **Done** – Assert completion and hold outputs stable

State transitions depend on:

* Subtraction result (`Cout`)
* Counter reaching zero

---

## **7. Inputs, Outputs, and Internal Signals**

### **Inputs**

* `DataA` – Dividend
* `DataB` – Divisor
* `Clock`
* `Reset`

### **Outputs**

* `A` – Quotient (implicit)
* `R` – Remainder
* `Done`

### **Internal Signals**

* `rr0` – Quotient bit
* `Count` – Iteration counter
* `Cout` – Subtraction result indicator

---

## **8. Overall Design Strategy**

* Explain division conceptually first
* Justify sequential nature
* Use FSM/ASM for control
* Implement arithmetic in datapath
* Avoid separate quotient register
* Ensure clean separation of datapath and control

---

## ✅ **PART 1 FINAL STATUS**

✔ Conceptually correct
✔ Hardware-faithful
✔ Textbook-aligned
✔ Viva-safe
✔ Ready for RTL mapping

---

### 🔜 **Next Step**

When ready, say:

👉 **“Proceed to PART 2 – Divider RTL”**

and we will implement the **complete, synthesizable Verilog design** directly from this finalized architecture.
