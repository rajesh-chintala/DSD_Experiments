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

---

# **7. Control Path Architecture**

The control path is responsible for **orchestrating datapath operations across clock cycles**. It generates control signals that enable loading, shifting, subtraction result selection, remainder update, quotient generation, and loop termination.

The control logic is implemented as an **Algorithmic State Machine (ASM)**, as shown in the FSM diagram.

---

## **7.1 Control Signals and Their Functions**

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

## **8. Inputs and Outputs**

---

### **Top-Level Divider Signals**

| Signal        | Description                                   |
| ------------- | --------------------------------------------- |
| `Clock`       | System clock controlling sequential operation |
| `Resetn`      | Active-low reset for FSM and datapath         |
| `s`           | Start signal to initiate division operation   |
| `LA`          | Load enable for dividend register A           |
| `EB`          | Load enable for divisor register B            |
| `DataA [7:0]` | Parallel input representing the dividend      |
| `DataB [7:0]` | Parallel input representing the divisor       |
| `Q [7:0]`     | Parallel output representing the quotient     |
| `R [7:0]`     | Parallel output representing the remainder    |
| `Done`        | Indicates completion of division operation    |

---

### **Internal Signals (Conceptual)**

| Signal       | Purpose                                                  |
| ------------ | -------------------------------------------------------- |
| `A`          | Dividend register initially; quotient register finally   |
| `B`          | Divisor register                                         |
| `R`          | Remainder register (upper bits)                          |
| `R0` / `rr0` | Flip-flop holding LSB of remainder                       |
| `Cout`       | Carry-out from subtraction indicating `R ≥ B` or `R < B` |
| `Sum`        | Result of 2’s complement subtraction                     |
| `y`          | Present FSM state                                        |
| `Y`          | Next FSM state                                           |
| `Count`      | Iteration counter                                        |
| `z`          | Counter zero flag                                        |
| `LR`         | Enables loading of remainder register                    |
| `EA`         | Enables shift of dividend/quotient register              |
| `ER`         | Enables shift of remainder register                      |
| `Rsel`       | Selects subtraction result or holds remainder            |

---


# **9. FSM / ASM State Description**

The FSM consists of **three main states**, with one **explicit shift-preparation step**, exactly as shown in the ASM diagram.

<img width="627" height="575" alt="image" src="https://github.com/user-attachments/assets/9fdfe072-614a-4752-9989-9c7819a3434a" />

---

## **9.1 State S1 — Initialization State**

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

## **9.2 Shift Preparation Step (EA, ER0)**

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

## **9.3 State S2 — Iterative Division State**

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

## **9.4 Remainder Update Control (`LR`)**

* The actual update of the remainder register occurs **only when `LR` is asserted**
* `LR` is enabled conditionally based on the result of `Cout`
* This ensures precise control over when subtraction results are committed

---

## **9.5 Counter Check and Decrement (z and EC)**

After the operations in **State S2**:

1. The FSM evaluates the zero flag `z`
2. **Only after this evaluation**, the counter is decremented using `EC`

**State transitions:**

* If `z = 0` → loop back to **S2**
* If `z = 1` → transition to **State S3**

This ordering is **explicitly shown in the FSM diagram** and is critical for correct operation.

---

## **9.6 State S3 — Done State**

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

## **9.7 FSM State–Based Control Signal Table**

The following table summarizes the **control signal behavior across FSM states**, clearly indicating which signals are asserted in each state.

---

### **FSM Control Signal Activity**

| Control Signal | **S1** (Initialization) | **Shift Prep** | **S2** (Iteration) | **S3** (Done) |
| -------------- | ----------------------- | -------------- | ------------------ | ------------- |
| `LA`           | 1                       | 0              | 0                  | 0             |
| `EB`           | 1                       | 0              | 0                  | 0             |
| `LR`           | 1                       | 0              | `Cout`             | 0             |
| `ER`           | 1                       | 0              | 1                  | 0             |
| `ER0`          | 0                       | 1              | 1                  | 0             |
| `EA`           | 0                       | 1              | 1                  | 0             |
| `Rsel`         | 0                       | 0              | 1                  | 0             |
| `LC`           | 1                       | 0              | 0                  | 0             |
| `EC`           | 0                       | 0              | 1*                 | 0             |
| `Done`         | 0                       | 0              | 0                  | 1             |

* `EC` is asserted **only after evaluating the zero flag (`z`)**.

---

# **10. Division Operation Explained Using FSM**

This section explains the division example shown in the table by correlating **FSM states**, **control sequencing**, and **datapath behavior**, with precise clock-cycle timing.

---

<img width="545" height="262" alt="image" src="https://github.com/user-attachments/assets/cad22d6f-74d2-4ccb-8d91-cf85710723d2" />

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

# **PART 2: SYNTHESIZABLE VERILOG RTL**

---

```verilog
//============================================================
// Sequential Binary Divider
// CONTROL PATH + DATAPATH (Textbook-Aligned)
//============================================================
module divider ( Clock, Resetn,s, LA, EB, DataA, DataB, Q, R, Done);

    parameter n = 8;
    parameter long = 3;

    input              Clock;
    input              Resetn;
    input              s;          // start signal
    input               LA;
    input               EB;
    input  [n-1:0]     DataA;       // dividend
    input  [n-1:0]     DataB;       // divisor
    output [n-1:0]     Q;           // quotient
    output [n-1:0]     R;           // remainder
    output reg         Done;

    //============================================================
    // INTERNAL SIGNAL DECLARATIONS
    //============================================================
    reg  [1:0] y, Y;

    reg  LR, ER, ER0;
    reg  EA;
    reg  Rsel;
    reg  LC, EC;

    wire [n-1:0] A, B;
    wire [n-1:0] DataR;
    wire [n:0]   Sum;
    wire         Cout;
    wire         R0;
    wire         z;
    wire [long-1:0] Count;

    //============================================================
    // CONTROL PATH
    //============================================================
    parameter S1 = 2'b00,
              S2 = 2'b01,
              S3 = 2'b10;

    always @(s, y, z) begin
        case (y)
            S1: if (s == 0) Y = S1;
                else        Y = S2;
            S2: if (z == 0) Y = S2;
                else        Y = S3;
            S3: if (s == 1) Y = S3;
                else        Y = S1;
            default: Y = S1;
        endcase
    end

    always @(posedge Clock or negedge Resetn) begin
        if (!Resetn)
            y <= S1;
        else
            y <= Y;
    end

    always @(y, Cout, z, s) begin
        LA = 0; EB = 0;
        LR = 0; ER = 0; ER0 = 0;
        EA = 0; Rsel = 0;
        LC = 0; EC = 0;
        Done = 0;

        case (y)
            S1: begin
                LA = 1; EB = 1; LC = 1; ER = 1;
                if (s == 0) begin
                    LR = 1; ER0 = 0;
                end else begin
                    EA = 1; ER0 = 1;
                end
            end

            S2: begin
                ER = 1; ER0 = 1; EA = 1; Rsel = 1;
                if (Cout) LR = 1;
                if (!z) EC = 1;
            end

            S3: begin
                Done = 1;
            end
        endcase
    end

    //============================================================
    // DATAPATH
    //============================================================
    regne   RegB   (DataB, Clock, Resetn, EB, B);
    defparam RegB.n = n;

    shiftln ShiftA (DataA, LA, EA, Cout, Clock, A);
    defparam ShiftA.n = n;
    assign Q = A;

    shiftln ShiftR (DataR, LR, ER, R0, Clock, R);
    defparam ShiftR.n = n;

    muxdff  FF_R0  (1'b0, A[n-1], ER0, Clock, R0);

    downcount Counter (Clock, EC, LC, Count);
    defparam Counter.n = long;
    assign z = (Count == 0);

    assign Sum  = {1'b0, R[n-2:0], R0} + {1'b0, ~B} + 1'b1;
    assign Cout = Sum[n];

    assign DataR = (Rsel) ? Sum[n-1:0] : 0;

endmodule
```

---

## **SUBMODULES **

---

### **Register with Enable (`regne`)**

```verilog
module regne (D, Clock, Resetn, E, Q);
    parameter n = 8;
    input  [n-1:0] D;
    input          Clock, Resetn, E;
    output reg [n-1:0] Q;

    always @(posedge Clock or negedge Resetn) begin
        if (!Resetn)
            Q <= 0;
        else if (E)
            Q <= D;
    end
endmodule
```

---

### **Left Shift Register with Load and Enable (`shiftln`)**

```verilog
module shiftln (D, L, E, w, Clock, Q);
    parameter n = 8;
    input  [n-1:0] D;
    input          L, E, w, Clock;
    output reg [n-1:0] Q;

    always @(posedge Clock) begin
        if (L)
            Q <= D;
        else if (E)
            Q <= {Q[n-2:0], w};
    end
endmodule
```

---

### **Multiplexed D Flip-Flop (`muxdff`)**

```verilog
module muxdff (d0, d1, s, Clock, Q);
    input d0, d1, s, Clock;
    output reg Q;

    always @(posedge Clock) begin
        if (s)
            Q <= d1;
        else
            Q <= d0;
    end
endmodule
```

---

### **Down Counter (`downcount`)**

```verilog
module downcount (Clock, E, L, Q);
    parameter n = 3;
    input Clock, E, L;
    output reg [n-1:0] Q;

    always @(posedge Clock) begin
        if (L)
            Q <= {n{1'b1}};
        else if (E)
            Q <= Q - 1'b1;
    end
endmodule
```

---

# **PART 3: TESTBENCH**

---

## **3.1 Testbench Objectives**

The objectives of this testbench are:

* To verify correct **functional behavior** of the sequential divider
* To validate **FSM sequencing and control**
* To confirm **quotient and remainder correctness**
* To observe **cycle-by-cycle evolution** of internal operations via outputs
* To ensure proper handling of:

  * Reset
  * Start signal
  * Completion (`Done`)

The testbench uses the **same example as explained in PART 1**.

---

## **3.2 Test Case Used (From PART 1 Example)**

| Parameter          | Value                    |
| ------------------ | ------------------------ |
| Dividend (`A`)     | `10001100` (decimal 140) |
| Divisor (`B`)      | `1001` (decimal 9)       |
| Expected Quotient  | `00001111` (decimal 15)  |
| Expected Remainder | `00000101` (decimal 5)   |

This is the **exact textbook example** used in the FSM + datapath explanation.

---

## **3.3 Testbench Strategy**

The testbench follows this sequence:

1. Apply **active-low reset**
2. Load dividend and divisor
3. Assert start signal `s`
4. Allow FSM to run for required cycles
5. Wait for `Done`
6. Observe final `Q` and `R`

---

## **3.4 Clock Generation**

* A free-running clock is generated
* Clock period is chosen to be **10 time units**
* No dependency on simulator-specific features

---

## **3.5 Reset and Start Handling**

* Reset is asserted at the beginning
* Reset is de-asserted before starting the division
* Start signal `s` is asserted **only once**
* FSM handles the rest of the sequencing

---

## **3.6 Divider Testbench Code**

```verilog
//============================================================
// Testbench for Sequential Divider
//============================================================
module tb_divider;

    // Parameters
    parameter n = 8;

    // Testbench signals
    reg              Clock;
    reg              Resetn;
    reg              s, LA, EB;
    reg  [n-1:0]     DataA;
    reg  [n-1:0]     DataB;
    wire [n-1:0]     Q;
    wire [n-1:0]     R;
    wire             Done;

    // Instantiate the Divider DUT
    divider #(.n(n)) DUT (
        .Clock (Clock),
        .Resetn(Resetn),
        .s     (s),
        .LA     (LA),
        .EB     (EB),
        .DataA (DataA),
        .DataB (DataB),
        .Q     (Q),
        .R     (R),
        .Done  (Done)
    );

    //============================================================
    // Clock Generation (10 time-unit period)
    //============================================================
    always #5 Clock = ~Clock;

    //============================================================
    // Test Sequence
    //============================================================
    initial begin
        // Initialize signals
        Clock  = 0;
        Resetn = 0;
        s      = 0;
        LA     = 1;
        EB     = 1;
        DataA  = 0;
        DataB  = 0;

        // Apply reset
        #10;
        Resetn = 1;

        // Load test operands (from PART 1 example)
        // Dividend = 10001100 (140)
        // Divisor  = 1001     (9)
        DataA = 8'b10001100;
        DataB = 8'b00001001;

        // Assert start signal, disable input loading
        #10;
        s = 1;
        LA = 0;
        EB = 0;

        // Deassert start after one cycle
        #10;
        s = 0;

        // Wait until division completes
        wait (Done == 1);

        // Display final results
        $display("====================================");
        $display("Division Completed");
        $display("Quotient  (Q) = %b", Q);
        $display("Remainder (R) = %b", R);
        $display("====================================");

        // End simulation
        #20;
        $finish;
    end

endmodule
```

---

## **3.7 Expected Simulation Outcome**

At the end of simulation:

* `Done = 1`
* `Q = 00001111` (15)
* `R = 00000101` (5)

These results **exactly match** the example explained in **PART 1, Section 9**.

---

## **3.8 Verification Observations**

* FSM transitions correctly from **S1 → S2 → S3**
* Quotient bits appear **one cycle after generation**
* Remainder updates only when `Cout = 1`
* Counter controls iteration count accurately
* Outputs remain stable after `Done` is asserted

---

# **PART 4: WAVEFORM & TIMING EXPLANATION**

---

<img width="1643" height="882" alt="image" src="https://github.com/user-attachments/assets/2034e44f-4bc8-4611-9f36-facf1e938d0a" />

## **4.1 Overview of the Observed Simulation**

The simulation corresponds to the division:

* **Dividend (`DataA`) = 10001100 (140)**
* **Divisor (`DataB`) = 00001001 (9)**

Final observed results:

* **Quotient (`Q`) = 00001111 (15)**
* **Remainder (`R`) = 00000101 (5)**
* **`Done` asserted after completion**

The waveform validates correct **FSM sequencing**, **datapath operation**, and **quotient/remainder timing**.

---

<img width="1642" height="876" alt="image" src="https://github.com/user-attachments/assets/0d600f6e-5cbc-48e6-ba08-cbce1198da32" />

## **4.2 Reset and Operand Loading Phase**

### **Reset Phase**

* `Resetn = 0`
* FSM is forced into **State S1 (Initialization)**
* Internal registers are cleared
* Outputs `Q` and `R` are invalid (`X`)

### **Operand Loading**

* `LA = 1`, `EB = 1`
* `DataA = 140`, `DataB = 9`
* On the next rising edge of `Clock`:

  * Register **A** loads the dividend
  * Register **B** loads the divisor
  * Register **R** and flip-flop **R0** are cleared

At this point:

* Register **A** holds the dividend
* Register **B** holds the divisor
* `R || R0 || A` is correctly initialized

---

## **4.3 Start Signal and FSM Transition**

* `s` is asserted for one clock cycle
* FSM transitions from **S1 → S2**
* `LA` and `EB` are deasserted
* Control is now fully FSM-driven

This marks the **beginning of the iterative division process**.

---

## **4.4 Clock Cycle 0: Shift Preparation and Initial Evaluation**

* A left shift of `R || R0 || A` is initiated
* MSB of **A** shifts into **R0**
* A subtraction is **already performed internally**:

  * `R − B` is evaluated using 2’s complement addition
  * Since `R = 0`, `R < B`
  * **`Cout = 0`**
* No remainder update occurs
* This explains why in the next cycle the action is labeled:

  * **Shift left, Q ← 0**

This cycle prepares alignment and establishes the **initial quotient bit = 0**.

---

## **4.5 Iterative Division Cycles (Cycles 1 to 3)**

For these cycles:

* `R || R0 || A` is shifted left
* Previous quotient bit (`Cout`) is inserted into LSB of **A**
* Subtraction is performed every cycle
* Observed behavior:

  * `R < B`
  * `Cout = 0`
  * `LR = 0` → remainder not updated
  * Quotient bits inserted are `0`

This matches the waveform where:

* `Q` evolves by left shifts with `0`s entering LSB
* `R` grows gradually but remains less than `B`

---

## **4.6 Critical Cycle: R Becomes Greater Than or Equal to B**

At the **fourth effective iteration**:

* After shifting, `R || R0` becomes **greater than or equal to `B`**
* Subtraction result is **positive**
* **`Cout = 1` is generated**
* FSM asserts:

  * `LR = 1` → subtraction result loaded into **R**
* **Important timing point**:

  * This quotient bit (`1`) is **generated now**
  * It is **inserted into A in the next clock cycle**

This explains why:

* The waveform shows the **quotient bit appearing one cycle later**

---

## **4.7 Subsequent Iterations (Cycles 5 to 8)**

For the remaining cycles:

* Each cycle:

  * Shift `R || R0 || A`
  * Insert previous `Cout` into LSB of **A**
  * Perform subtraction
* Observed:

  * `R ≥ B`
  * `Cout = 1`
  * `LR = 1` → remainder updated
  * Quotient bits inserted are `1`

This results in:

* `Q` converging to `00001111`
* `R` converging to `00000101`

---

## **4.8 Counter, Zero Flag, and Completion**

* The down-counter decrements after each iteration
* When `Count == 0`:

  * `z = 1`
  * FSM transitions **S2 → S3**
* In **State S3**:

  * `Done = 1`
  * `Q` and `R` are stable and valid

This matches the waveform where:

* `Done` asserts only after final quotient and remainder are ready

---

## **4.9 Key Timing Insights**

* **Subtraction is performed every cycle**, regardless of outcome
* `Cout` simultaneously:

  * Indicates `R ≥ B` or `R < B`
  * Generates the quotient bit
* **Quotient generation and insertion are separated by one cycle**
* `R` is conditionally updated using `LR`
* FSM ensures **exactly `n` iterations**

---

## **4.10 Final Verification Summary**

| Signal         | Observed Result | Expected  |
| -------------- | --------------- | --------- |
| `Q`            | `00001111`      | ✔ Correct |
| `R`            | `00000101`      | ✔ Correct |
| `Done`         | Asserted        | ✔ Correct |
| FSM sequencing | S1 → S2 → S3    | ✔ Correct |

<img width="1640" height="876" alt="image" src="https://github.com/user-attachments/assets/41aed57c-19c8-4c77-b37f-b5c600e99f59" />

---

# **PART 5: VIVA, INTERVIEW, DEBUG & DESIGN INSIGHTS**

---

## **A) Viva Questions (University / Lab-Oriented)**

1. Why is division inherently a **sequential operation**, unlike addition?
2. What is the role of the **FSM** in your divider design?
3. Why is subtraction performed **in every iteration**, even when `R < B`?
4. What is the significance of the signal `Cout` in this design?
5. Why is the quotient bit **generated in one cycle and inserted in the next cycle**?
6. Explain the purpose of splitting the remainder into **R and R0**.
7. Why is register **A used both as dividend and quotient**?
8. How does the FSM ensure that the division runs for **exactly `n` iterations**?
9. What happens if the start signal `s` is held high continuously?
10. Why are `LA` and `EB` controlled externally in your implementation?
11. What ensures that the outputs `Q` and `R` are valid only when `Done = 1`?
12. How does your design relate to the **binary long-division method**?

---

## **B) Interview-Style Questions (VLSI / Industry)**

1. How would you modify this divider to support **signed division**?
2. What are the area and timing trade-offs between:

   * Restoring division
   * Non-restoring division
3. How would you **pipeline** this divider to improve throughput?
4. Why is a **down-counter** preferred over comparing iteration count manually?
5. How would you parameterize this design for different bit-widths?
6. What would happen if `Cout` were sampled combinationally instead of synchronously?
7. How would this divider behave if integrated into a **CPU datapath**?
8. How do you ensure this design is **synthesis-safe**?
9. What verification strategy would you use for corner cases like:

   * Dividend < divisor
   * Dividend = 0
10. How would you formally verify correctness of this divider?

---

## **C) Debug Scenarios (Real RTL Issues)**

### **Bug 1: Blocking Assignment in FSM Register**

```verilog
always @(posedge Clock)
    y = Y;   // Wrong
```

**Problem:**

* Race conditions
* Simulation vs synthesis mismatch
* Unstable FSM behavior

**Fix:**

```verilog
always @(posedge Clock)
    y <= Y;
```

---

### **Bug 2: Quotient Bit Appears One Cycle Late**

**Observation:**

* Designer expects quotient to update immediately

**Reason:**

* Quotient is **generated from `Cout`**
* Inserted only during the **next shift**

**Conclusion:**

* This is **correct behavior**, not a bug

---

### **Bug 3: Remainder Always Updates**

**Cause:**

* `LR` asserted unconditionally

**Fix:**

* Assert `LR` **only when `Cout = 1`**

---

### **Bug 4: Division Never Completes**

**Cause:**

* Counter not decremented after `z` evaluation

**Fix:**

* Decrement counter **only after checking `z`**

---

## **D) Design Variations**

1. **Parallel Divider**

   * Faster but area-intensive
2. **Non-Restoring Divider**

   * Reduced cycles, more complex control
3. **Signed Divider**

   * Requires sign preprocessing and postprocessing
4. **Handshake-Based Divider**

   * Start/Done replaced by valid/ready
5. **Parameterized Divider**

   * Supports arbitrary bit-widths

---

## **E) Conceptual Reasoning Questions**

1. Why is separating **datapath and control path** important?
2. What advantages does an FSM provide over ad-hoc control logic?
3. How does this design improve clarity compared to a monolithic RTL block?
4. Why is the quotient accumulated MSB-to-LSB?
5. What would break if subtraction were not performed every cycle?

---

## **F) If This Were in a Real Chip…**

### **Power**

* Clock gating on:

  * Registers
  * Counter
* Disable logic after `Done`

### **Clocking**

* Single-clock domain
* Clean synchronous control

### **CDC**

* Required if `s` comes from another clock domain

### **Verification Strategy**

* Directed tests
* Random constrained tests
* Assertion: `Done → outputs stable`
* Coverage on:

  * `Cout`
  * FSM transitions
  * Counter reaching zero

---

# **Part-6: Schematics:**

---

## **RTL Schematic**
---
<img width="1644" height="879" alt="image" src="https://github.com/user-attachments/assets/be223fc0-28af-49e0-8d1a-c3a2cb296f3e" />

## **Technology Schematic**
---
<img width="1641" height="879" alt="image" src="https://github.com/user-attachments/assets/c758b4b4-0212-49bf-bf93-1aada0c5471e" />
<img width="1641" height="882" alt="image" src="https://github.com/user-attachments/assets/61b640a1-f24d-4853-8ad3-e4cf3138e807" />
<img width="1640" height="882" alt="image" src="https://github.com/user-attachments/assets/844267a9-76c0-492e-a406-3dad30ccd07c" />

---

# ** Part-7: Constraints:**
---
```
create_clock -period 10 -name Clock [get_ports Clock]

set_property PACKAGE_PIN E3 [get_ports Clock]
set_property IOSTANDARD LVCMOS33 [get_ports Clock]

set_property PACKAGE_PIN V10 [get_ports {DataA[7]}]
set_property PACKAGE_PIN U11 [get_ports {DataA[6]}]
set_property PACKAGE_PIN U12 [get_ports {DataA[5]}]
set_property PACKAGE_PIN H6 [get_ports {DataA[4]}]
set_property PACKAGE_PIN T13 [get_ports {DataA[3]}]

set_property PACKAGE_PIN R16 [get_ports {DataA[2]}]
set_property PACKAGE_PIN U8 [get_ports {DataA[1]}]
set_property PACKAGE_PIN T8 [get_ports {DataA[0]}]
set_property PACKAGE_PIN R13 [get_ports {DataB[7]}]
set_property PACKAGE_PIN U18 [get_ports {DataB[6]}]
set_property PACKAGE_PIN T18 [get_ports {DataB[5]}]
set_property PACKAGE_PIN R17 [get_ports {DataB[4]}]
set_property PACKAGE_PIN R15 [get_ports {DataB[3]}]
set_property PACKAGE_PIN M13 [get_ports {DataB[2]}]
set_property PACKAGE_PIN L16 [get_ports {DataB[1]}]
set_property PACKAGE_PIN J15 [get_ports {DataB[0]}]
set_property PACKAGE_PIN V11 [get_ports {Q[7]}]
set_property PACKAGE_PIN V12 [get_ports {Q[6]}]
set_property PACKAGE_PIN V14 [get_ports {Q[5]}]
set_property PACKAGE_PIN V15 [get_ports {Q[4]}]
set_property PACKAGE_PIN T16 [get_ports {Q[3]}]
set_property PACKAGE_PIN U14 [get_ports {Q[2]}]
set_property PACKAGE_PIN T15 [get_ports {Q[1]}]
set_property PACKAGE_PIN V16 [get_ports {Q[0]}]
set_property PACKAGE_PIN U16 [get_ports {R[7]}]
set_property PACKAGE_PIN U17 [get_ports {R[6]}]
set_property PACKAGE_PIN V17 [get_ports {R[5]}]
set_property PACKAGE_PIN R18 [get_ports {R[4]}]
set_property PACKAGE_PIN N14 [get_ports {R[3]}]
set_property PACKAGE_PIN J13 [get_ports {R[2]}]
set_property PACKAGE_PIN K15 [get_ports {R[1]}]
set_property PACKAGE_PIN H17 [get_ports {R[0]}]
set_property PACKAGE_PIN P17 [get_ports EB]
set_property PACKAGE_PIN M17 [get_ports LA]
set_property PACKAGE_PIN M18 [get_ports Resetn]
set_property PACKAGE_PIN P18 [get_ports s]
set_property PACKAGE_PIN N16 [get_ports Done]
set_property IOSTANDARD LVCMOS33 [get_ports Done]
set_property IOSTANDARD LVCMOS33 [get_ports EB]
set_property IOSTANDARD LVCMOS33 [get_ports LA]
set_property IOSTANDARD LVCMOS33 [get_ports Resetn]
set_property IOSTANDARD LVCMOS33 [get_ports s]
set_property IOSTANDARD LVCMOS33 [get_ports {DataA[7]}]
set_property IOSTANDARD LVCMOS33 [get_ports {DataA[6]}]
set_property IOSTANDARD LVCMOS33 [get_ports {DataA[5]}]
set_property IOSTANDARD LVCMOS33 [get_ports {DataA[4]}]
set_property IOSTANDARD LVCMOS33 [get_ports {DataA[3]}]
set_property IOSTANDARD LVCMOS33 [get_ports {DataA[2]}]
set_property IOSTANDARD LVCMOS33 [get_ports {DataA[1]}]
set_property IOSTANDARD LVCMOS33 [get_ports {DataA[0]}]
set_property IOSTANDARD LVCMOS33 [get_ports {DataB[7]}]
set_property IOSTANDARD LVCMOS33 [get_ports {DataB[6]}]
set_property IOSTANDARD LVCMOS33 [get_ports {DataB[5]}]
set_property IOSTANDARD LVCMOS33 [get_ports {DataB[4]}]
set_property IOSTANDARD LVCMOS33 [get_ports {DataB[3]}]
set_property IOSTANDARD LVCMOS33 [get_ports {DataB[2]}]
set_property IOSTANDARD LVCMOS33 [get_ports {Q[7]}]
set_property IOSTANDARD LVCMOS33 [get_ports {Q[6]}]
set_property IOSTANDARD LVCMOS33 [get_ports {Q[5]}]
set_property IOSTANDARD LVCMOS33 [get_ports {Q[4]}]
set_property IOSTANDARD LVCMOS33 [get_ports {DataB[1]}]
set_property IOSTANDARD LVCMOS33 [get_ports {DataB[0]}]
set_property IOSTANDARD LVCMOS33 [get_ports {Q[2]}]
set_property IOSTANDARD LVCMOS33 [get_ports {Q[1]}]
set_property IOSTANDARD LVCMOS33 [get_ports {Q[0]}]
set_property IOSTANDARD LVCMOS33 [get_ports {R[7]}]
set_property IOSTANDARD LVCMOS33 [get_ports {R[6]}]
set_property IOSTANDARD LVCMOS33 [get_ports {R[5]}]
set_property IOSTANDARD LVCMOS33 [get_ports {R[4]}]
set_property IOSTANDARD LVCMOS33 [get_ports {R[3]}]
set_property IOSTANDARD LVCMOS33 [get_ports {R[2]}]
set_property IOSTANDARD LVCMOS33 [get_ports {R[1]}]
set_property IOSTANDARD LVCMOS33 [get_ports {R[0]}]
set_property DRIVE 12 [get_ports {Q[3]}]
set_property DRIVE 12 [get_ports {Q[2]}]
set_property DRIVE 12 [get_ports {Q[1]}]
set_property DRIVE 12 [get_ports {Q[0]}]
set_property DRIVE 12 [get_ports {R[7]}]
set_property DRIVE 12 [get_ports {R[6]}]
set_property DRIVE 12 [get_ports {R[5]}]
set_property DRIVE 12 [get_ports {R[4]}]
set_property DRIVE 12 [get_ports {R[3]}]
set_property DRIVE 12 [get_ports {R[2]}]
set_property DRIVE 12 [get_ports {R[1]}]
set_property DRIVE 12 [get_ports {R[0]}]
set_property SLEW SLOW [get_ports {Q[3]}]
set_property SLEW SLOW [get_ports {Q[2]}]
set_property SLEW SLOW [get_ports {Q[1]}]
set_property SLEW SLOW [get_ports {Q[0]}]
set_property SLEW SLOW [get_ports {R[7]}]
set_property SLEW SLOW [get_ports {R[6]}]
set_property SLEW SLOW [get_ports {R[5]}]
set_property SLEW SLOW [get_ports {R[4]}]
set_property SLEW SLOW [get_ports {R[3]}]
set_property SLEW SLOW [get_ports {R[2]}]
set_property SLEW SLOW [get_ports {R[1]}]
set_property SLEW SLOW [get_ports {R[0]}]
set_property IOSTANDARD LVCMOS33 [get_ports {Q[3]}]

```
---

# Final Project Summary
<img width="1641" height="853" alt="image" src="https://github.com/user-attachments/assets/70800999-4623-4749-9041-9694f36f5163" />
<img width="1642" height="849" alt="image" src="https://github.com/user-attachments/assets/58bd68b0-cf74-4362-8e94-e6dc583bd7cd" />
<img width="824" height="163" alt="image" src="https://github.com/user-attachments/assets/04613158-f27d-4e73-9c18-84a960e195c2" />


