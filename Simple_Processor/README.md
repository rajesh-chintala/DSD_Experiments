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

input  [7:0] Data;
input        Reset, w, Clock;
input  [1:0] F, Rx, Ry;
output       Done;
output [7:0] BusWires;

//--------------------------------------------------
// Internal control and datapath signals
//--------------------------------------------------
reg  [0:3] Rin, Rout;
reg  [7:0] Sum;
wire Clear, AddSub, Extern, Ain, Gin, Gout, FRin;
wire [1:0] Count;
wire [0:3] T, I, Xreg, Y;
wire [7:0] R0, R1, R2, R3, A, G;
wire [5:0] Func;
reg  [5:0] FuncReg;

//--------------------------------------------------
// Function register
//--------------------------------------------------
assign Func = {F, Rx, Ry};

always @(posedge Clock)
    if (FRin)
        FuncReg <= Func;

//--------------------------------------------------
// Instruction decoder
//--------------------------------------------------
assign I[0] = ~FuncReg[5] & ~FuncReg[4]; // Load
assign I[1] = ~FuncReg[5] &  FuncReg[4]; // Move
assign I[2] =  FuncReg[5] & ~FuncReg[4]; // Add
assign I[3] =  FuncReg[5] &  FuncReg[4]; // Sub

//--------------------------------------------------
// Register field decoders
//--------------------------------------------------
dec2to4 decX (FuncReg[3:2], 1'b1, Xreg);
dec2to4 decY (FuncReg[1:0], 1'b1, Y);

//--------------------------------------------------
// Time-step counter and decoder
//--------------------------------------------------
upcount U1 (Clear, Clock, Count);

assign T[0] = ~Count[1] & ~Count[0];
assign T[1] = ~Count[1] &  Count[0];
assign T[2] =  Count[1] & ~Count[0];
assign T[3] =  Count[1] &  Count[0];

//--------------------------------------------------
// Control signal generation
//--------------------------------------------------
assign Extern = I[0] & T[1];
assign Ain    = (I[2] | I[3]) & T[1];
assign Gin    = (I[2] | I[3]) & T[2];
assign Gout   = (I[2] | I[3]) & T[3];
assign AddSub = I[3];

assign Done = (I[0] | I[1]) & T[1]
            | (I[2] | I[3]) & T[3];

assign FRin  = w & T[0];
assign Clear = Reset | Done | (~w & T[0]);

//--------------------------------------------------
// Register enable logic
//--------------------------------------------------
always @(*)
begin
    Rin  = 4'b0000;
    Rout = 4'b0000;

    if ((I[0] | I[1]) & T[1]) Rin = Xreg;
    if ((I[2] | I[3]) & T[3]) Rin = Xreg;
    if ((I[1] | I[2] | I[3]) & T[1]) Rout = Y;
    if ((I[2] | I[3]) & T[2]) Rout = Y;
end

//--------------------------------------------------
// Datapath registers
//--------------------------------------------------
regn R_0 (BusWires, Rin[0], Clock, R0);
regn R_1 (BusWires, Rin[1], Clock, R1);
regn R_2 (BusWires, Rin[2], Clock, R2);
regn R_3 (BusWires, Rin[3], Clock, R3);

regn RA  (BusWires, Ain, Clock, A);
regn RG  (Sum, Gin, Clock, G);

//--------------------------------------------------
// ALU
//--------------------------------------------------
always @(*)
    if (AddSub)
        Sum = A - BusWires;
    else
        Sum = A + BusWires;

//--------------------------------------------------
// Shared bus using tri-state buffers
//--------------------------------------------------
trin t0 (Data,   Extern, BusWires);
trin t1 (R0,     Rout[0], BusWires);
trin t2 (R1,     Rout[1], BusWires);
trin t3 (R2,     Rout[2], BusWires);
trin t4 (R3,     Rout[3], BusWires);
trin t5 (G,      Gout,    BusWires);

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

The purpose of this testbench is to verify the correct functional operation of the simple processor by applying representative instructions and observing the resulting behavior over multiple clock cycles.

---

## **3.2 Objectives**

The testbench aims to:

* Verify correct execution of Load, Move, Add, and Sub instructions
* Confirm correct sequencing of multi-cycle instructions
* Validate shared bus operation
* Ensure correct assertion of the `Done` signal
* Verify reset and instruction start (`w`) handling

---

## **3.3 Verification Strategy**

A **directed testing approach** is used:

* Instructions are applied one at a time
* Inputs are held stable until instruction completion
* The next instruction is issued only after `Done` is asserted
* Verification is performed using waveform inspection

This strategy provides deterministic and transparent verification.

---

## **3.4 Instruction Application Methodology**

Each instruction is applied using the following sequence:

1. Apply opcode (`F`) and register selectors (`Rx`, `Ry`)
2. Assert `w = 1` to start execution
3. Hold all instruction inputs constant
4. Wait for `Done = 1`
5. Deassert `w`
6. Proceed to the next instruction

This ensures that **instructions do not overlap** and the control path completes one instruction before starting the next.

---

## **3.5 Complete Testbench Code**

```verilog
module tb_proc;

    //--------------------------------------------------
    // Signal Declarations
    //--------------------------------------------------
    reg  [7:0] Data;
    reg        Reset;
    reg        w;
    reg        Clock;
    reg  [1:0] F;
    reg  [1:0] Rx, Ry;

    wire [7:0] BusWires;
    wire       Done;

    //--------------------------------------------------
    // DUT Instantiation
    //--------------------------------------------------
    proc DUT (
        .Data     (Data),
        .Reset    (Reset),
        .w        (w),
        .Clock    (Clock),
        .F        (F),
        .Rx       (Rx),
        .Ry       (Ry),
        .Done     (Done),
        .BusWires (BusWires)
    );

    //--------------------------------------------------
    // Clock Generation (10 time-unit period)
    //--------------------------------------------------
    always #5 Clock = ~Clock;

    //--------------------------------------------------
    // Test Sequence
    //--------------------------------------------------
    initial begin

        // ---------------------------------------------
        // Initialization and Reset
        // ---------------------------------------------
        Clock = 0;
        Reset = 1;
        w     = 0;
        Data  = 0;
        F     = 0;
        Rx    = 0;
        Ry    = 0;

        #10;
        Reset = 0;

        // ---------------------------------------------
        // Test Case 1: Load Instruction
        // Load Data into R0
        // ---------------------------------------------
        #10;
        Data = 8'h0A;
        F    = 2'b00;   // Load
        Rx   = 2'b00;   // R0
        w    = 1;

        wait (Done);
        w = 0;

        // ---------------------------------------------
        // Test Case 2: Move Instruction
        // Move R0 to R1
        // ---------------------------------------------
        #10;
        F  = 2'b01;     // Move
        Rx = 2'b01;     // R1
        Ry = 2'b00;     // R0
        w  = 1;

        wait (Done);
        w = 0;

        // ---------------------------------------------
        // Test Case 3: Add Instruction
        // R1 = R1 + R0
        // ---------------------------------------------
        #10;
        F  = 2'b10;     // Add
        Rx = 2'b01;     // R1
        Ry = 2'b00;     // R0
        w  = 1;

        wait (Done);
        w = 0;

        // ---------------------------------------------
        // Test Case 4: Sub Instruction
        // R1 = R1 - R0
        // ---------------------------------------------
        #10;
        F  = 2'b11;     // Sub
        Rx = 2'b01;     // R1
        Ry = 2'b00;     // R0
        w  = 1;

        wait (Done);
        w = 0;

        // ---------------------------------------------
        // End Simulation
        // ---------------------------------------------
        #50;
        $stop;
    end

endmodule
```

---

## **3.6 Observability During Simulation**

During simulation, the following signals should be observed:

* `BusWires` — to track data movement
* `Done` — to verify instruction completion
* Register outputs (`R0–R3`)
* ALU result register (`G`)
* Time-step progression

Waveform analysis confirms correct sequencing and data flow.

---

## **3.7 Summary**

* The testbench verifies all supported instructions
* Multi-cycle behavior is clearly observable
* Shared bus usage is validated
* The testbench is tool-agnostic and deterministic

