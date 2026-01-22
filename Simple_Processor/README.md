# **PART 1: THEORY, ARCHITECTURE & CONTROL DESIGN**

---

## **1. Concept / Theory of a Simple Processor**

A **processor** is a digital system that executes instructions by performing a sequence of simple operations such as moving data, storing data, and performing arithmetic.

Each instruction does not execute all at once. Instead, it is broken into **small steps**, and each step is performed in a **separate clock cycle**. This step-by-step execution allows us to clearly understand **how data moves inside the processor**.

The processor implemented in this experiment is intentionally kept **simple** so that students can clearly observe:

* How instructions are executed internally
* How registers and buses are used
* Why control signals are required

---

## **2. Instruction Set and Supported Operations**

The processor supports a small instruction set that is sufficient to demonstrate internal processor operation.

### **Supported Instructions**

| Instruction     | Meaning                                       |
| --------------- | --------------------------------------------- |
| `Load Rx, Data` | Load external data into register `Rx`         |
| `Move Rx, Ry`   | Copy data from register `Ry` to register `Rx` |
| `Add Rx, Ry`    | Add contents of `Ry` to `Rx`                  |
| `Sub Rx, Ry`    | Subtract contents of `Ry` from `Rx`           |

These instructions allow us to study:

* External data loading
* Register-to-register transfer
* Arithmetic operations
* Multi-cycle execution

---

## **3. Why the Processor Operates Sequentially**

This processor is designed to operate **sequentially**, meaning instructions take multiple clock cycles.

The main reasons are:

* There is **only one shared bus**
* Only **one data transfer** can occur per clock cycle
* Arithmetic operations require multiple steps:

  * Operand preparation
  * Computation
  * Result storage

Because of these limitations, each instruction is divided into **time steps**, and each step occurs on a clock edge. This sequential behavior makes the internal operation **easy to observe and understand**.

---

# **4. DATAPATH ARCHITECTURE**

The **datapath** is the part of the processor that actually handles data. It does not decide *what* to do; it only performs actions when enabled by control signals.

---

## **4.1 Datapath Overview**

The datapath consists of the following components:

* General-purpose registers (`R0–R3`)
* Operand register (`A`)
* Result register (`G`)
* Arithmetic Logic Unit (ALU)
* Tri-state buffers
* A shared bus
* External data input

All these components are connected in such a way that **data can move through the processor in a controlled manner**.

---

## **4.2 General-Purpose Registers (R0–R3)**

The processor contains four general-purpose registers used to store operands and results.

Each register has two important control signals:

* **`Rout`** – enables the register to place its contents on the bus
* **`Rin`** – enables the register to load data from the bus

### **Operation**

* When `Rout = 1`, the register drives the bus
* When `Rin = 1`, the register captures the bus value at the clock edge

Only **one `Rout` signal is allowed to be active at a time**, ensuring controlled data transfer.

---

## **4.3 Tri-State Buffer (Datapath Component)**

A **tri-state buffer** is a special circuit that can place its output in one of three states:

* Logic `0`
* Logic `1`
* High-impedance (`Z`)

### **Role in the Processor**

* Each register output connects to the bus through a tri-state buffer
* The buffer is enabled using signals such as `Rout`, `Gout`, or `Extern`
* When disabled, the buffer disconnects the register from the bus

### **Why Tri-State Buffers Are Used**

* They allow multiple sources to share a single bus
* They prevent electrical contention
* They make data movement explicit and observable

---

## **4.4 Shared Bus**

The **shared bus** is the central communication path of the datapath.

* It carries data between registers and functional units
* It is driven by **exactly one tri-state buffer at a time**
* All other sources remain disconnected

The bus simplifies wiring and clearly shows how data flows during instruction execution.

---

## **4.5 Register A – Operand Register**

Register `A` is used to store the **first operand** for arithmetic operations.

Control signal:

* **`Ain`** – loads the bus value into register A

### **Purpose**

* Provides a stable operand to the ALU
* Prevents the ALU input from changing when the bus changes
* Enables multi-cycle arithmetic execution

---

## **4.6 Arithmetic Logic Unit (ALU)**

The ALU performs:

* Addition
* Subtraction

Inputs:

* One operand from register `A`
* One operand from the bus

The operation is selected using a control signal such as:

* **`AddSub`** (0 → Add, 1 → Subtract)

The ALU output is not written directly to a register.

---

## **4.7 Register G – Result Register**

Register `G` stores the output of the ALU.

Control signals:

* **`Gin`** – loads ALU output into `G`
* **`Gout`** – places `G` contents onto the bus

### **Why Register G Is Needed**

* Only one bus transfer is allowed per cycle
* Computation and write-back must occur in different cycles
* `G` allows clean separation of these steps

---

# **5. Control Path Architecture**

The **control path** governs the sequencing and coordination of datapath operations. It determines **which control signals are asserted in each clock cycle** so that instruction execution proceeds in a correct and orderly manner.

The control path in this processor is built using:

* A **function register**
* **Opcode and register field decoding**
* A **time-step generator**
* Control signal generation logic

Together, these elements implement a structured, multi-cycle instruction execution mechanism.
---

## **5.1 Control Signals Used in the Processor**

The control path generates the following control signals to coordinate datapath operation and instruction sequencing.

---

### **Instruction and Sequencing Control**

* **`w`** – Instruction start/control signal
* **`FRin`** – Loads the instruction into the Function Register
* **`Done`** – Indicates completion of the current instruction
* **`Clear`** – Clears the time-step counter (returns to initial step)

---

### **Time-Step Signals**

* **`T0`** – Initial execution step
* **`T1`** – First operation step
* **`T2`** – Second operation step
* **`T3`** – Final operation step

(Only one time-step signal is active at a time.)

---

### **Bus Control Signals**

* **`Rout`** – Enables the selected general-purpose register to drive the bus
* **`Gout`** – Enables register `G` (ALU result) to drive the bus
* **`Extern`** – Enables external data input to drive the bus

---

### **Register Load Control Signals**

* **`Rin`** – Loads data from the bus into the selected general-purpose register
* **`Ain`** – Loads the operand register `A`
* **`Gin`** – Loads the ALU output into register `G`

---

### **ALU Control Signal**

* **`AddSub`** – Selects ALU operation

  * `0` → Addition
  * `1` → Subtraction

---

## **5.2 Function Register (FR)**

The **Function Register (FR)** stores the instruction information required by the control path during execution.

### **Purpose of the Function Register**

* Holds instruction fields stable across multiple clock cycles
* Provides inputs to opcode and register decoders
* Separates instruction loading from instruction execution

---

### **Contents of the Function Register**

The function register contains:

| Field    | Description                          |
| -------- | ------------------------------------ |
| `f1, f2` | Opcode bits defining the instruction |
| `Rx`     | Destination register field           |
| `Ry`     | Source register field                |

The contents of the function register are decoded continuously while the instruction is being executed.

---

## **5.3 Opcode Interpretation Using `f1` and `f2`**

The opcode bits `f1` and `f2` uniquely identify the instruction.

| `f1` | `f2` | Operation |
| ---- | ---- | --------- |
| 0    | 0    | Load      |
| 0    | 1    | Move      |
| 1    | 0    | Add       |
| 1    | 1    | Sub       |

These opcode bits are decoded to determine **which control actions must occur** during each time step.

This approach avoids explicit instruction naming and keeps the control logic compact and systematic.

---

## **5.4 Register Field Decoding (`Rx`, `Ry`)**

The register fields select which registers participate in the instruction.

### **Destination Register Selection (`Rx`)**

The `Rx` field is decoded to generate register input-enable signals:

| `Rx` | Active Register Input |
| ---- | --------------------- |
| 00   | `R0in`                |
| 01   | `R1in`                |
| 10   | `R2in`                |
| 11   | `R3in`                |

This determines **which register captures data from the bus**.

---

### **Source Register Selection (`Ry`)**

The `Ry` field is decoded to generate register output-enable signals:

| `Ry` | Active Register Output |
| ---- | ---------------------- |
| 00   | `R0out`                |
| 01   | `R1out`                |
| 10   | `R2out`                |
| 11   | `R3out`                |

This determines **which register drives the shared bus**.

---

## **5.5 Time-Step Generator**

Instruction execution is divided into **time steps**, with each step occupying one clock cycle.

### **Time-Step Signals**

| Signal | Meaning               |
| ------ | --------------------- |
| `T0`   | First execution step  |
| `T1`   | Second execution step |
| `T2`   | Third execution step  |
| `T3`   | Fourth execution step |

Only one time-step signal is active at any time.

---

### **Hardware Used for Time-Step Generation**

The time steps are generated using:

* A **2-bit up-counter**
* A **2-to-4 decoder**
* 
---

## **5.6 Control Signal Assertion Summary (Operation vs Time Step)**

This section explains **which control signals are asserted at each time step**, instruction-wise, based on the control behavior summarized in the given figure.

---

### **(Load) Instruction — `I0`**

**Time Step: `T1`**

* `Extern` → External data drives the bus
* `Rin = X` → Destination register `Rx` loads data
* `Done` → Instruction completes

**Key Points**

* No ALU operation involved
* Single-cycle instruction
* Direct data transfer from external input to register

---

### **(Move) Instruction — `I1`**

**Time Step: `T1`**

* `Rout = Y` → Source register `Ry` drives the bus
* `Rin = X` → Destination register `Rx` captures data
* `Done` → Instruction completes

**Key Points**

* Register-to-register transfer
* No arithmetic operation
* Single-cycle execution

---

### **(Add) Instruction — `I2`**

**Time Step: `T1`**

* `Rout = X` → Destination register places operand on bus
* `Ain` → Operand register `A` captures first operand

**Time Step: `T2`**

* `Rout = Y` → Source register places second operand on bus
* `Gin` → ALU result stored in register `G`
* `AddSub = 0` → ALU performs addition

**Time Step: `T3`**

* `Gout` → ALU result drives the bus
* `Rin = X` → Result written back to destination register
* `Done` → Instruction completes

**Key Points**

* Three-cycle instruction
* Separate operand fetch, compute, and write-back phases
* ALU configured for addition

---

### **(Sub) Instruction — `I3`**

**Time Step: `T1`**

* `Rout = X` → Destination register places operand on bus
* `Ain` → Operand register `A` captures first operand

**Time Step: `T2`**

* `Rout = Y` → Source register places second operand on bus
* `Gin` → ALU result stored in register `G`
* `AddSub = 1` → ALU performs subtraction

**Time Step: `T3`**

* `Gout` → ALU result drives the bus
* `Rin = X` → Result written back to destination register
* `Done` → Instruction completes

**Key Points**

* Identical sequencing to Add instruction
* Only difference is ALU operation selection
* Multi-cycle arithmetic execution

---

### **Overall Observations**

* **Load and Move** → single-cycle instructions
* **Add and Sub** → three-cycle instructions
* Only **one bus driver enabled per time step**
* Clear separation of:

  * Operand fetch
  * ALU operation
  * Result write-back

---

## **5.7 Control Signal Generation Logic (Combinational Expressions)**

All control signals are generated using **pure combinational logic** based on:

* Decoded instruction signals: `I0` (Load), `I1` (Move), `I2` (Add), `I3` (Sub)
* Time-step signals: `T0`, `T1`, `T2`, `T3`
* Register selection fields: `Rx`, `Ry`

At any time, **only one instruction signal (`I0–I3`) is active**.

---

### **Bus and Register Control**

```
Extern = I0 · T1
Ain    = (I2 + I3) · T1
Gin    = (I2 + I3) · T2
Gout   = (I2 + I3) · T3
```

---

### **ALU Control**

```
AddSub = I3
```

---

### **Register Enable Signals**

```
Rout = (I1 + I2 + I3) · (T1 + T2) · Ry
Rin  = [ (I0 + I1) · T1 + (I2 + I3) · T3 ] · Rx
```

---

### **Instruction and Sequencing Control**

```
Done = (I0 + I1) · T1 + (I2 + I3) · T3
FRin = w · T0
```

---

### **Time-Step Counter Control**

```
Clear = Reset | Done | (~w & T0)
```

---

## **5.8 FSM Interpretation of the Control Path**

Although the control path does not use an explicit FSM description, its behavior is equivalent to a **finite state machine**.

| FSM Concept      | Implementation           |
| ---------------- | ------------------------ |
| State            | Time step (`T0–T3`)      |
| State register   | Counter                  |
| State transition | Counter increment        |
| Output logic     | Control signal equations |

Thus, the processor uses an **implicit FSM**, implemented using a counter and combinational logic.

---

## **5.9 Instruction Completion**

When the final time step of an instruction is reached:

* The `Done` signal is asserted
* The counter is cleared
* Control signals are deasserted
* The processor is ready for the next instruction

This guarantees **non-overlapping instruction execution**.

---
Below are the **two additional sections for PART 1**, written in the **same professional, textbook-style tone**, concise but explanatory, and **fully consistent with the processor design discussed so far**.

---

## **6. Inputs and Outputs**

This section summarizes the external and internal interfaces of the processor, clarifying how data and control interact with the system.
---

## **Processor Input Signals**

| Signal       | Description                          |
| ------------ | ------------------------------------ |
| `Data [7:0]` | Parallel external data input         |
| `Clock`      | System clock                         |
| `Reset`      | Active-high reset                    |
| `w`          | Instruction execution control signal |
| `F [1:0]`    | Opcode field (instruction type)      |
| `Rx [1:0]`   | Destination register selector        |
| `Ry [1:0]`   | Source register selector             |

---

## **Processor Output Signals**

| Signal           | Description                                   |
| ---------------- | --------------------------------------------- |
| `BusWires [7:0]` | Shared internal data bus                      |
| `Done`           | Indicates completion of instruction execution |

---

## **Internal Datapath Registers**

| Signal     | Description                |
| ---------- | -------------------------- |
| `R0 [7:0]` | General-purpose register 0 |
| `R1 [7:0]` | General-purpose register 1 |
| `R2 [7:0]` | General-purpose register 2 |
| `R3 [7:0]` | General-purpose register 3 |
| `A [7:0]`  | Operand register           |
| `G [7:0]`  | ALU result register        |

---

## **Internal Control and Status Signals**

| Signal        | Description                       |
| ------------- | --------------------------------- |
| `Rin [0:3]`   | Register input enable signals     |
| `Rout [0:3]`  | Register output enable signals    |
| `T [0:3]`     | One-hot time-step signals         |
| `I [0:3]`     | Decoded instruction signals       |
| `Xreg [0:3]`  | Decoded destination register      |
| `Y [0:3]`     | Decoded source register           |
| `FRin`        | Function register load enable     |
| `Extern`      | External data bus enable          |
| `Ain`         | Operand register load enable      |
| `Gin`         | ALU result register load enable   |
| `Gout`        | ALU result register output enable |
| `AddSub`      | ALU operation select              |
| `Clear`       | Time-step counter clear           |
| `Count [1:0]` | Time-step counter output          |

---

## **7. Overall Processor Architecture Description**

The processor follows a **clear separation between datapath and control path**, which simplifies design, understanding, and verification.

---

### **7.1 Datapath Summary**

The datapath consists of:

* A set of general-purpose registers (`R0–R3`)
* Operand register `A`
* Result register `G`
* An adder/subtractor (ALU)
* A shared bus implemented using tri-state buffers

The datapath is responsible for:

* Storing operands and results
* Moving data via the shared bus
* Performing arithmetic operations

All datapath actions occur **only when enabled by control signals**.

---

### **7.2 Control Path Summary**

The control path consists of:

* Function Register (FR)
* Opcode and register-field decoders
* Time-step counter and decoder
* Combinational control logic

The control path:

* Interprets the instruction stored in the Function Register
* Generates time-step signals
* Asserts appropriate control signals at each time step
* Ensures correct sequencing of datapath operations

---

### **7.3 Datapath–Control Path Interaction**

Processor operation proceeds as follows:

1. An instruction is loaded into the Function Register.
2. The control path decodes the instruction and determines required execution steps.
3. Time-step signals sequence the instruction over multiple clock cycles.
4. Control signals activate datapath components in the correct order.
5. The instruction completes when `Done` is asserted.

This structured interaction ensures:

* Deterministic execution
* No bus contention
* Clear correspondence between instruction semantics and hardware behavior

---

# **PART 2: SYNTHESIZABLE VERILOG RTL**

---

## **2.1 Design Requirements**

The processor RTL satisfies the following requirements:

* Multi-cycle instruction execution using a shared bus
* Control implemented using:

  * Function register
  * Opcode decoding
  * Time-step counter and decoder
* Datapath consisting of:

  * General-purpose registers
  * Operand and result registers
  * Adder/Subtractor ALU
* Shared bus implemented using tri-state buffers
* Fully synchronous design
* Synthesizable using standard FPGA tools

---

## **2.2 Module Interfaces**

---

## **Top-Level Processor Module (`proc`)**

### **Input Signals**

| Signal       | Description                   |
| ------------ | ----------------------------- |
| `Data [7:0]` | External data input           |
| `Clock`      | System clock                  |
| `Reset`      | Active-high reset             |
| `w`          | Instruction execution control |
| `F [1:0]`    | Opcode field                  |
| `Rx [1:0]`   | Destination register selector |
| `Ry [1:0]`   | Source register selector      |

---

### **Output Signals**

| Signal           | Description                      |
| ---------------- | -------------------------------- |
| `BusWires [7:0]` | Shared internal data bus         |
| `Done`           | Instruction completion indicator |

---

## **n-Bit Register Module (`regn`)**

### **Input Signals**

| Signal      | Description  |
| ----------- | ------------ |
| `R [n-1:0]` | Data input   |
| `L`         | Load enable  |
| `Clock`     | System clock |

### **Output Signals**

| Signal      | Description     |
| ----------- | --------------- |
| `Q [n-1:0]` | Register output |

---

## **Two-Bit Up-Counter Module (`upcount`)**

### **Input Signals**

| Signal  | Description           |
| ------- | --------------------- |
| `Clear` | Counter reset control |
| `Clock` | System clock          |

### **Output Signals**

| Signal    | Description    |
| --------- | -------------- |
| `Q [1:0]` | Counter output |

---

## **2-to-4 Decoder Module (`dec2to4`)**

### **Input Signals**

| Signal    | Description   |
| --------- | ------------- |
| `w [1:0]` | Binary input  |
| `En`      | Enable signal |

### **Output Signals**

| Signal    | Description            |
| --------- | ---------------------- |
| `y [0:3]` | One-hot decoded output |

---

## **Tri-State Buffer Module (`trin`)**

### **Input Signals**

| Signal      | Description   |
| ----------- | ------------- |
| `Y [n-1:0]` | Data input    |
| `E`         | Enable signal |

### **Output Signals**

| Signal      | Description      |
| ----------- | ---------------- |
| `F [n-1:0]` | Tri-state output |

---

## **2.3 Submodule RTL Code **

---

### **2.3.1 n-Bit Register**

```verilog
module regn (R, L, Clock, Q);
    parameter n = 8;
    input  [n-1:0] R;
    input          L, Clock;
    output reg [n-1:0] Q;

    always @(posedge Clock)
        if (L)
            Q <= R;
endmodule
```

---

### **2.3.2 2-to-4 Decoder**

```verilog
module dec2to4 (w, En, y);
    input  [1:0] w;
    input        En;
    output reg [0:3] y;

    always @(*)
        if (En)
            case (w)
                2'b00: y = 4'b1000;
                2'b01: y = 4'b0100;
                2'b10: y = 4'b0010;
                2'b11: y = 4'b0001;
            endcase
        else
            y = 4'b0000;
endmodule
```

---

### **2.3.3 Two-Bit Up Counter**

```verilog
module upcount (Clear, Clock, Q);
    input        Clear, Clock;
    output reg [1:0] Q;

    always @(posedge Clock)
        if (Clear)
            Q <= 2'b00;
        else
            Q <= Q + 1'b1;
endmodule
```

---

### **2.3.4 Tri-State Buffer**

```verilog
module trin (Y, E, F);
    parameter n = 8;
    input  [n-1:0] Y;
    input          E;
    output wire [n-1:0] F;

    assign F = E ? Y : 'bz;
endmodule
```

---

## **2.4 Top-Level Processor RTL**

```verilog
module proc (Data, Reset, w, Clock, F, Rx, Ry, Done, BusWires);

    //--------------------------------------------------
    // Top-Level Input and Output Ports
    //--------------------------------------------------
    input  [7:0] Data;          // External data input
    input        Reset, w;      // Reset and instruction start control
    input        Clock;         // System clock
    input  [1:0] F;             // Opcode field
    input  [1:0] Rx, Ry;        // Register selection fields
    output wire [7:0] BusWires; // Shared internal bus
    output       Done;          // Instruction completion signal

    //--------------------------------------------------
    // Internal Control and Datapath Signals
    //--------------------------------------------------
    reg  [0:3] Rin, Rout;       // Register input/output enables
    reg  [7:0] Sum;             // ALU output (combinational)

    wire Clear, AddSub;          // Control signals
    wire Extern, Ain, Gin, Gout; // Datapath control enables
    wire FRin;                   // Function register load
    wire [1:0] Count;            // Time-step counter output
    wire [0:3] T, I;             // Time-step and instruction decode
    wire [0:3] Xreg, Y;          // Register decode signals
    wire [7:0] R0, R1, R2, R3;   // General-purpose registers
    wire [7:0] A, G;             // Operand and result registers
    wire [1:6] Func, FuncReg;    // Instruction fields and stored instruction

    integer k;                   // Loop variable for register control

    //--------------------------------------------------
    // Timing Control: Counter and Time-Step Decoder
    //--------------------------------------------------
    upcount counter (Clear, Clock, Count); // 2-bit time-step counter
    dec2to4 decT (Count, 1'b1, T);         // Decode time steps T0–T3

    // Counter clear logic and function register control
    assign Clear = Reset | Done | (~w & T[0]);
    assign Func  = {F, Rx, Ry};             // Combine instruction fields
    assign FRin  = w & T[0];                // Load instruction at T0

    //--------------------------------------------------
    // Function Register
    //--------------------------------------------------
    regn functionreg (Func, FRin, Clock, FuncReg);
        defparam functionreg.n = 6;

    //--------------------------------------------------
    // Instruction and Register Field Decoding
    //--------------------------------------------------
    dec2to4 decI (FuncReg[1:2], 1'b1, I);   // Instruction decode
    dec2to4 decX (FuncReg[3:4], 1'b1, Xreg);// Destination register decode
    dec2to4 decY (FuncReg[5:6], 1'b1, Y);   // Source register decode

    //--------------------------------------------------
    // Control Signal Generation
    //--------------------------------------------------
    assign Extern = I[0] & T[1];   // External data enable (Load)
    assign Done   = ((I[0] | I[1]) & T[1]) |
                    ((I[2] | I[3]) & T[3]); // Instruction completion
    assign Ain    = (I[2] | I[3]) & T[1];    // Load operand register A
    assign Gin    = (I[2] | I[3]) & T[2];    // Load ALU result register G
    assign Gout   = (I[2] | I[3]) & T[3];    // Drive ALU result on bus
    assign AddSub = I[3];                    // ALU operation select

    //--------------------------------------------------
    // Register Input and Output Control
    //--------------------------------------------------
    always @(I, T, Xreg, Y)
        for (k = 0; k < 4; k = k + 1)
        begin
            // Register input enable logic
            Rin[k]  = ((I[0] | I[1]) & T[1] & Xreg[k]) |
                      ((I[2] | I[3]) & T[3] & Xreg[k]);

            // Register output enable logic
            Rout[k] = (I[1] & T[1] & Y[k]) |
                      ((I[2] | I[3]) &
                      ((T[1] & Xreg[k]) | (T[2] & Y[k])));
        end

    //--------------------------------------------------
    // Datapath: Shared Bus and Registers
    //--------------------------------------------------
    trin tri_ext (Data, Extern, BusWires);   // External data to bus

    regn reg_0 (BusWires, Rin[0], Clock, R0);
    regn reg_1 (BusWires, Rin[1], Clock, R1);
    regn reg_2 (BusWires, Rin[2], Clock, R2);
    regn reg_3 (BusWires, Rin[3], Clock, R3);

    trin tri_0 (R0, Rout[0], BusWires);
    trin tri_1 (R1, Rout[1], BusWires);
    trin tri_2 (R2, Rout[2], BusWires);
    trin tri_3 (R3, Rout[3], BusWires);

    regn reg_A (BusWires, Ain, Clock, A);    // Operand register A

    //--------------------------------------------------
    // Arithmetic Logic Unit (ALU)
    //--------------------------------------------------
    always @(AddSub, A, BusWires)
        if (!AddSub)
            Sum = A + BusWires;  // Addition
        else
            Sum = A - BusWires;  // Subtraction

    //--------------------------------------------------
    // Result Register and Bus Interface
    //--------------------------------------------------
    regn reg_G (Sum, Gin, Clock, G);         // Result register G
    trin tri_G (G, Gout, BusWires);           // Drive result on bus

endmodule

```

---

## **2.5 Design Characteristics**

* Multi-cycle processor with implicit FSM control
* Shared bus architecture using tri-state buffers
* Clear separation of datapath and control path
* Deterministic instruction execution using time steps
* Minimal hardware with educational clarity
* Fully synthesizable and simulation-safe RTL

---

# **PART 3: TESTBENCH (VERIFICATION)**

---

## **3.1 Purpose of the Testbench**

The purpose of this testbench is to verify the correct functional operation of the simple processor by applying a sequence of instructions and observing their execution behavior over time.
The testbench ensures that instruction decoding, control sequencing, and datapath operations work together correctly.

---

## **3.2 Objectives**

The objectives of this testbench are:

* To verify correct execution of all supported instructions:

  * Load
  * Move
  * Add
  * Sub
* To validate instruction sequencing using the control signal `w`
* To confirm correct assertion of the `Done` signal
* To verify proper shared-bus operation
* To ensure correct processor behavior after reset

---

## **3.3 Verification Strategy**

A **directed, step-by-step verification strategy** is used:

* Each instruction is applied independently
* The processor is reset before each instruction
* The instruction start signal (`w`) is asserted for one clock pulse
* Inputs remain stable during instruction execution
* Instruction completion is observed via `Done`
* Waveform inspection is used for validation

This strategy simplifies debugging and clearly isolates instruction behavior.

---

## **3.4 Instruction Application Methodology**

Each instruction is applied using the following sequence:

1. Assert `Reset` to initialize the processor
2. Apply instruction opcode (`F`) and register selectors (`Rx`, `Ry`)
3. Deassert `Reset`
4. Assert `w` for one clock cycle to start execution
5. Deassert `w`
6. Allow sufficient time for instruction completion

Resetting before each instruction ensures **clean and independent verification**.

---

## **3.5 Complete Testbench Code**

```verilog
module tb_proc();

    //--------------------------------------------------
    // Testbench Signal Declarations
    //--------------------------------------------------
    reg  [7:0] Data;
    reg        Reset, w, Clock;
    reg  [1:0] F, Rx, Ry;
    wire [7:0] BusWires;
    wire       Done;

    //--------------------------------------------------
    // DUT Instantiation
    //--------------------------------------------------
    proc dut (
        Data, Reset, w, Clock, F, Rx, Ry, Done, BusWires
    );

    //--------------------------------------------------
    // Clock Generation (10 time-unit period)
    //--------------------------------------------------
    initial begin
        Clock = 1'b0;
        forever #5 Clock = ~Clock;
    end

    //--------------------------------------------------
    // Test Sequence
    //--------------------------------------------------
    initial begin

        //--------------------------------------------------
        // Test Case 1: Load Instruction
        // Load Data = 10 into R0
        //--------------------------------------------------
        Reset = 1;
        w     = 0;
        F     = 2'b00;     // Load
        Data  = 8'd10;
        Rx    = 2'b00;     // R0
        Ry    = 2'b00;

        #10 Reset = 0;
        #10 w = 1;
        #10 w = 0;

        //--------------------------------------------------
        // Test Case 2: Move Instruction
        // Move R0 to R1
        //--------------------------------------------------
        #10 Reset = 1;
        F  = 2'b01;        // Move
        Ry = 2'b00;        // R0
        Rx = 2'b01;        // R1

        #10 Reset = 0;
        #10 w = 1;
        #10 w = 0;

        //--------------------------------------------------
        // Test Case 3: Add Instruction
        // R1 = R1 + R0
        //--------------------------------------------------
        #10 Reset = 1;
        F  = 2'b10;        // Add

        #10 Reset = 0;
        #10 w = 1;
        #10 w = 0;

        //--------------------------------------------------
        // Test Case 4: Sub Instruction
        // R1 = R1 - R0
        //--------------------------------------------------
        #40 Reset = 1;
        F  = 2'b11;        // Sub

        #10 Reset = 0;
        #10 w = 1;
        #10 w = 0;

        //--------------------------------------------------
        // End Simulation
        //--------------------------------------------------
        #100;
        $finish;
    end

endmodule
```

---

## **3.6 Applied Inputs and Expected Outputs**

---

### **Instruction 1: Load**

**Applied Inputs**

* `F = 00`
* `Rx = 00`
* `Data = 10`
* `w = 1`

**Expected Behavior**

* External data drives `BusWires`
* Register `R0` loads value `10`
* Instruction completes in one cycle

---

### **Instruction 2: Move**

**Applied Inputs**

* `F = 01`
* `Ry = 00`
* `Rx = 01`
* `w = 1`

**Expected Behavior**

* `R0` drives the bus
* `R1` captures value `10`
* Instruction completes in one cycle

---

### **Instruction 3: Add**

**Applied Inputs**

* `F = 10`
* `Rx = 01`
* `Ry = 00`
* `w = 1`

**Expected Behavior**

* ALU computes `R1 + R0`
* Result written back to `R1`
* Multi-cycle execution

---

### **Instruction 4: Sub**

**Applied Inputs**

* `F = 11`
* `Rx = 01`
* `Ry = 00`
* `w = 1`

**Expected Behavior**

* ALU computes `R1 - R0`
* Result written back to `R1`
* Multi-cycle execution

---

## **3.7 Observability During Simulation**

The following signals should be observed in the waveform:

* `BusWires`
* `Done`
* Register outputs (`R0`, `R1`)
* Operand register `A`
* Result register `G`
* Time-step signals (`T0–T3`)

Correct sequencing and data movement confirm functional correctness.

---

## **3.8 Summary**

* All instructions are verified independently
* Reset-based isolation simplifies debugging
* Control-path sequencing is clearly observable
* The testbench is deterministic and tool-agnostic


<img width="1435" height="879" alt="image" src="https://github.com/user-attachments/assets/83bc63d4-34cc-49f0-92ac-5c8ce70e6ace" />

<img width="1627" height="881" alt="image" src="https://github.com/user-attachments/assets/6e6f5fe2-ad39-46b8-a4dc-41b71e29a9df" />

<img width="1617" height="878" alt="image" src="https://github.com/user-attachments/assets/87f10027-58dc-41ca-984b-9cad0f935fda" />

<img width="1433" height="877" alt="image" src="https://github.com/user-attachments/assets/0b12c5c4-e918-47c6-8ed2-7f0fc9c47ce5" />

<img width="1918" height="1041" alt="image" src="https://github.com/user-attachments/assets/f02a22f2-d896-4d59-808e-5edc574ab520" />

<img width="1920" height="1041" alt="image" src="https://github.com/user-attachments/assets/7fc175ab-f571-4bb5-91eb-6cb200f00469" />

<img width="1920" height="1040" alt="image" src="https://github.com/user-attachments/assets/027d4e75-dff0-4b33-af16-a548f987b719" />

---

# **PART 4: WAVEFORM & TIMING EXPLANATION**

---

## **4.1 Overview of Observed Waveform**

The waveform demonstrates the execution of four instructions in sequence:

1. **Load**
2. **Move**
3. **Add**
4. **Sub**

Each instruction is initiated by asserting `w` and completes when `Done` is asserted.
The processor operates in **multiple time steps (T0–T3)** controlled by the internal time-step counter.

Key signals observed:

* `Clock`
* `Reset`
* `w`
* `F`, `Rx`, `Ry`
* `BusWires`
* `Done`
* Internal registers (`R0`, `R1`, `A`, `G`, `Sum`)

---

## **4.2 Clock and Reset Behavior**

* The clock is a **free-running periodic signal** with a constant period.
* `Reset` is asserted **before each instruction**:

  * Clears the time-step counter
  * Resets control sequencing
  * Ensures independent instruction execution

This reset-before-instruction approach simplifies verification and ensures deterministic behavior.

---

## **4.3 Instruction Start and Completion (`w` and `Done`)**

* `w` is asserted for **one clock cycle** to initiate instruction execution.
* When `w = 1` at **T0**:

  * Instruction fields `{F, Rx, Ry}` are loaded into the function register.
* `Done` is asserted:

  * At **T1** for single-cycle instructions (Load, Move)
  * At **T3** for multi-cycle instructions (Add, Sub)

Once `Done = 1`, the instruction is complete and the time-step counter is cleared.

---

## **4.4 Load Instruction Timing**

**Observed Opcode:** `F = 00`

### **Operation**

* External data is loaded into a general-purpose register.

### **Waveform Interpretation**

* At `T1`:

  * `Extern = 1`
  * `BusWires` carries the value from `Data`
  * Destination register (`R0`) loads the value
* `Done` is asserted in the same cycle

### **Result**

* `R0` updates to `00001010` (decimal 10)
* Instruction completes in **one clock cycle**

---

## **4.5 Move Instruction Timing**

**Observed Opcode:** `F = 01`

### **Operation**

* Data is transferred from source register to destination register.

### **Waveform Interpretation**

* At `T1`:

  * Source register (`R0`) drives `BusWires`
  * Destination register (`R1`) captures the value
* `Done` is asserted at `T1`

### **Result**

* `R1` becomes `00001010`
* No ALU involvement
* Single-cycle execution confirmed

---

## **4.6 Add Instruction Timing**

**Observed Opcode:** `F = 10`

### **Operation**

* `R1 = R1 + R0`

### **Cycle-by-Cycle Explanation**

**T1**

* `Rout = X` → Destination register value placed on bus
* `Ain = 1` → Operand register `A` loads value

**T2**

* `Rout = Y` → Source register drives bus
* `Gin = 1` → ALU performs addition
* Result stored in register `G`

**T3**

* `Gout = 1` → ALU result placed on bus
* `Rin = X` → Result written back to destination register
* `Done = 1`

### **Result**

* `Sum` and `G` show correct addition
* `R1` updates to `00010100` (decimal 20)
* Multi-cycle behavior clearly visible

---

## **4.7 Sub Instruction Timing**

**Observed Opcode:** `F = 11`

### **Operation**

* `R1 = R1 − R0`

### **Cycle-by-Cycle Explanation**

**T1**

* Operand register `A` loads destination register value

**T2**

* ALU performs subtraction (`AddSub = 1`)
* Result stored in `G`

**T3**

* ALU result written back to destination register
* `Done` asserted

### **Result**

* `R1` returns to `00001010` (decimal 10)
* Subtraction verified correctly

---

## **4.8 Shared Bus Behavior**

The waveform clearly shows:

* `BusWires` driven by **only one source at a time**
* High-impedance (`Z`) when no source is enabled
* Proper tri-state arbitration from:

  * External data
  * Register outputs
  * ALU result register

This confirms correct **tri-state bus control**.

---

## **4.9 Control Path and Datapath Synchronization**

The waveform demonstrates tight coordination between:

* **Control path**

  * Instruction decoding
  * Time-step sequencing
  * Control signal assertion
* **Datapath**

  * Register loading
  * ALU operation
  * Bus transfers

Each instruction progresses deterministically through the expected time steps.

---

## **4.10 Summary of Observations**

* Single-cycle instructions complete at `T1`
* Multi-cycle instructions complete at `T3`
* `Done` accurately marks instruction completion
* Bus conflicts are avoided
* ALU operations are correctly sequenced
* Reset ensures clean instruction boundaries

---

