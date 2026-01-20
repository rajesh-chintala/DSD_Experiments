# **PART 1: THEORY, ARCHITECTURE, FSM DESIGN**

---

## **1. Concept / Theory**

A **serial adder** is a sequential digital circuit that performs binary addition by processing **one bit per clock cycle**. The addition starts from the **least significant bit (LSB)** and proceeds toward the **most significant bit (MSB)**. Instead of using multiple full adders, a serial adder uses a **single full-adder operation repeatedly**, along with storage elements to retain intermediate results.

This approach significantly reduces hardware complexity and is suitable for systems where **area and power efficiency** are prioritized over speed.

---

### **Serial Adder vs Parallel Adder**

| Parameter          | Parallel Adder      | Serial Adder             |
| ------------------ | ------------------- | ------------------------ |
| Hardware resources | High                | Low                      |
| Speed              | Fast (single cycle) | Slower (multiple cycles) |
| Area               | Large               | Small                    |
| Power consumption  | Higher              | Lower                    |
| Operation          | All bits at once    | One bit per cycle        |

---

## **2. Need for Sequential Operation**

In serial addition:

* Each bit addition depends on the **carry generated from the previous bit**
* The carry must be **stored across clock cycles**
* This introduces **memory and time dependency**

As a result, a serial adder is inherently a **sequential circuit**, not a purely combinational one.

---

## **3. Role of Finite State Machine (FSM)**

A **finite state machine** is used to model and control the carry behavior in the serial adder.

* The FSM stores the **carry value**
* The current state represents the **carry-in**
* The next state represents the **carry-out**
* FSM ensures deterministic and structured carry propagation

Using an FSM simplifies the design and makes carry handling explicit and reliable.

---

## **4. FSM Model Selection**

The serial adder is implemented using a **Mealy-type FSM**, where:

* Outputs depend on:

  * Present state (carry)
  * Present inputs (current bits)
* The sum output is generated **within the same clock cycle**
* State updates occur only on the clock edge

This model provides lower latency for sum generation compared to a Moore-based approach.

---

## **5. Inputs and Outputs**

### **Top-Level Serial Adder Signals**

<img width="509" height="208" alt="image" src="https://github.com/user-attachments/assets/c9841665-d733-4631-9709-1650965e0f7e" />


| Signal      | Description              |
| ----------- | ------------------------ |
| `A [7:0]`   | Parallel input operand A |
| `B [7:0]`   | Parallel input operand B |
| `Clock`     | System clock             |
| `Reset`     | Active-high reset        |
| `Sum [7:0]` | Parallel output sum      |

---

### **Internal Signals (Conceptual)**

| Signal  | Purpose                     |
| ------- | --------------------------- |
| `QA[0]` | Current bit of operand A    |
| `QB[0]` | Current bit of operand B    |
| `s`     | Current sum bit             |
| `y`     | Present carry state         |
| `Y`     | Next carry state            |
| `Run`   | Controls shifting operation |

---

## **6. State Definition and Meaning**

The FSM consists of **two states**, representing the carry condition.

| State | Meaning   |
| ----- | --------- |
| `G`   | Carry = 0 |
| `H`   | Carry = 1 |

Only one flip-flop is required to store the FSM state, making the design minimal and efficient.

---

## **7. Full-Adder Logic Basis**

The serial adder operation is based on standard full-adder behavior.

<img width="404" height="249" alt="image" src="https://github.com/user-attachments/assets/8f5b002b-b5ea-4341-87b1-430fe529b049" />

* **Sum** depends on:

  * Input bit from A
  * Input bit from B
  * Carry-in (FSM state)
* **Carry-out** is determined by the same inputs and becomes the next FSM state

This direct correspondence ensures correct bit-by-bit addition.

---

## **8. State Transition Behavior**

<img width="434" height="215" alt="image" src="https://github.com/user-attachments/assets/cb43122c-b48c-4c08-b588-19340412f5eb" />


### **When Carry = 0 (State G)**

| A bit | B bit | Sum | Next State |
| ----- | ----- | --- | ---------- |
| 0     | 0     | 0   | G          |
| 0     | 1     | 1   | G          |
| 1     | 0     | 1   | G          |
| 1     | 1     | 0   | H          |

---

### **When Carry = 1 (State H)**

| A bit | B bit | Sum | Next State |
| ----- | ----- | --- | ---------- |
| 0     | 0     | 1   | G          |
| 0     | 1     | 0   | H          |
| 1     | 0     | 0   | H          |
| 1     | 1     | 1   | H          |

---

## **9. Textual FSM Representation**

```
State G (Carry = 0)
  A B = 00 → Sum = 0 → G
  A B = 01 → Sum = 1 → G
  A B = 10 → Sum = 1 → G
  A B = 11 → Sum = 0 → H

State H (Carry = 1)
  A B = 00 → Sum = 1 → G
  A B = 01 → Sum = 0 → H
  A B = 10 → Sum = 0 → H
  A B = 11 → Sum = 1 → H
```

---

## **10. Overall Architecture Description**

The serial adder architecture consists of:

* **Input shift registers** for operands A and B
* **FSM-based carry logic**
* **Output shift register** to collect sum bits
* **Counter-based control logic** to limit operation to operand width

This separation of datapath and control ensures clarity, scalability, and reliable operation.

---

## **11. Design Strategy**

* Use shift registers to supply one bit per cycle
* Use FSM to retain carry across cycles
* Generate sum using combinational logic
* Update carry synchronously
* Control operation length using a counter
* Initialize all elements using reset

# **PART 2: SYNTHESIZABLE VERILOG RTL**

---

## **1. Design Requirements Recap**

The serial adder RTL is designed to meet the following requirements:

* Perform addition of two multi-bit binary numbers in a **serial manner**
* Process **one bit per clock cycle**
* Preserve carry information using a **finite state machine**
* Use **shift registers** to handle serial data flow
* Generate sum using **Mealy-type output behavior**
* Terminate operation automatically after all bits are processed
* Remain **fully synthesizable** and suitable for simulation and implementation

---

## **2. Interface Definition**

### **Top-Level Serial Adder Module**

**Inputs**

* `A [7:0]` : Parallel input operand A
* `B [7:0]` : Parallel input operand B
* `Reset`   : Active-high reset signal
* `Clock`   : System clock

**Output**

* `Sum [7:0]` : Parallel sum output

---

### **Shift Register Module**

**Inputs**

* `R [n-1:0]` : Parallel data input
* `L`         : Load control
* `E`         : Shift enable
* `w`         : Serial input bit
* `Clock`     : Clock

**Output**

* `Q [n-1:0]` : Register contents

---

## **3. FSM Encoding and Meaning**

The finite state machine maintains the **carry state** of the serial adder.

| State | Meaning      |
| ----- | ------------ |
| `G`   | Carry-in = 0 |
| `H`   | Carry-in = 1 |

* Only a **single flip-flop** is required
* The FSM state directly represents the carry memory
* This ensures minimal hardware and clear state interpretation

---

## **4. Shift Register RTL**

The shift register supports:

* Parallel loading of data
* Controlled shifting on each clock cycle
* Serial insertion of bits

```verilog
module shiftreg (R, L, E, w, Clock, Q);

    parameter n = 8;

    input  [n-1:0] R;
    input          L;
    input          E;
    input          w;
    input          Clock;
    output reg [n-1:0] Q;

    integer k;

    always @(posedge Clock) begin
        if (L)
            Q <= R;              // Parallel load
        else if (E) begin
            for (k = n-1; k > 0; k = k-1)
                Q[k-1] <= Q[k];  // Shift operation
            Q[n-1] <= w;         // Insert serial bit
        end
    end

endmodule
```

---

## **5. Serial Adder RTL**

This module integrates the datapath and control logic required for serial addition.

```verilog
module serial_adder (A, B, Reset, Clock, Sum);

    input  [7:0] A, B;
    input        Reset, Clock;
    output [7:0] Sum;

    reg  [3:0] Count;
    reg  s, y, Y;
    wire [7:0] QA, QB;
    wire Run;

    // FSM state encoding
    parameter G = 1'b0, H = 1'b1;

    // Shift registers for operands and sum
    shiftreg shift_A   (A, Reset, 1'b1, 1'b0, Clock, QA);
    shiftreg shift_B   (B, Reset, 1'b1, 1'b0, Clock, QB);
    shiftreg shift_Sum (8'b0, Reset, Run, s, Clock, Sum);

    // ----------------------------------------
    // FSM: Output and Next-State Logic
    // ----------------------------------------
    always @(QA, QB, y) begin
        case (y)
            G: begin
                s = QA[0] ^ QB[0];
                if (QA[0] & QB[0])
                    Y = H;
                else
                    Y = G;
            end

            H: begin
                s = QA[0] ^ ~QB[0];
                if (~QA[0] & ~QB[0])
                    Y = G;
                else
                    Y = H;
            end

            default: Y = G;
        endcase
    end

    // ----------------------------------------
    // FSM State Register
    // ----------------------------------------
    always @(posedge Clock) begin
        if (Reset)
            y <= G;
        else
            y <= Y;
    end

    // ----------------------------------------
    // Counter-Based Control Logic
    // ----------------------------------------
    always @(posedge Clock) begin
        if (Reset)
            Count <= 4'd8;
        else if (Run)
            Count <= Count - 1'b1;
    end

    assign Run = |Count;

endmodule
```

---

## **6. Operational Description**

* On reset:

  * Input operands are loaded into shift registers
  * Carry state is initialized
  * Counter is set to the operand width
* On each clock cycle:

  * One bit from each operand is processed
  * Sum bit is generated immediately
  * Carry is updated via the FSM
  * Sum bit is shifted into the output register
* When all bits are processed:

  * Counter reaches zero
  * Shifting operation stops

---

## **7. Design Characteristics**

* Clear separation between datapath and control
* Minimal FSM with explicit carry representation
* No delays or non-synthesizable constructs
* Predictable timing behavior
* Suitable for simulation, synthesis, and hardware realization

---

## **8. Summary of PART 2**

* Complete serial adder RTL
* FSM-based carry handling
* Shift-register-based serial data flow
* Deterministic and verifiable operation
* Ready for downstream verification and implementation

---

# **PART 4: WAVEFORM & TIMING EXPLANATION (CYCLE-BY-CYCLE)**

---

## **1. Overview of Timing Behavior**

* The serial adder processes **one bit per clock cycle**
* Operands are shifted **LSB first**
* Carry is stored in the FSM state
* The output sum is accumulated using a shift register
* Operation continues until the counter reaches zero

---

## **2. Signals Considered**

| Signal     | Description               |
| ---------- | ------------------------- |
| `Clock`    | System clock              |
| `Reset`    | Active-high reset         |
| `QA[0]`    | Current bit of operand A  |
| `QB[0]`    | Current bit of operand B  |
| `y`        | FSM present state (carry) |
| `Y`        | FSM next state            |
| `s`        | Sum bit generated         |
| `Count`    | Remaining cycles          |
| `Run`      | Shift enable              |
| `Sum[7:0]` | Output sum register       |

---

## **3. Reset and Initialization (Cycle 0)**

* `Reset = 1`
* Parallel operands loaded:

  * `QA = 00001011`
  * `QB = 00001101`
* FSM carry initialized to `0`
* `Count = 8`
* Output `Sum` is not yet valid

---

## **4. Cycle-by-Cycle Operation**

### **Input Operands**

* A = `00001011` (11)
* B = `00001101` (13)

---

### **Cycle 1**

| Parameter        | Value      |
| ---------------- | ---------- |
| `QA[0]`          | 1          |
| `QB[0]`          | 1          |
| Carry-in (`y`)   | 0          |
| `s`              | 0          |
| Next Carry (`Y`) | 1          |
| `Sum`            | `00000000` |

---

### **Cycle 2**

| Parameter  | Value      |
| ---------- | ---------- |
| `QA[0]`    | 1          |
| `QB[0]`    | 0          |
| Carry-in   | 1          |
| `s`        | 0          |
| Next Carry | 1          |
| `Sum`      | `00000000` |

---

### **Cycle 3**

| Parameter  | Value      |
| ---------- | ---------- |
| `QA[0]`    | 0          |
| `QB[0]`    | 1          |
| Carry-in   | 1          |
| `s`        | 0          |
| Next Carry | 1          |
| `Sum`      | `00000000` |

---

### **Cycle 4**

| Parameter  | Value      |
| ---------- | ---------- |
| `QA[0]`    | 1          |
| `QB[0]`    | 1          |
| Carry-in   | 1          |
| `s`        | 1          |
| Next Carry | 1          |
| `Sum`      | `00000000` |

---

### **Cycle 5**

| Parameter      | Value      |
| -------------- | ---------- |
| Shifted-in bit | 1          |
| `Sum`          | `10000000` |

---

### **Cycle 6**

| Parameter      | Value      |
| -------------- | ---------- |
| Shifted-in bit | 1          |
| `Sum`          | `11000000` |

---

### **Cycle 7**

| Parameter      | Value      |
| -------------- | ---------- |
| Shifted-in bit | 0          |
| `Sum`          | `01100000` |

---

### **Cycle 8**

| Parameter      | Value      |
| -------------- | ---------- |
| Shifted-in bit | 0          |
| `Sum`          | `00011000` |

---

## **5. End of Operation**

* `Count = 0`
* `Run = 0`
* Shifting stops automatically
* Output remains stable

Final result:

```
Sum = 00011000 (24)
```

---

## **6. FSM State Progression**

| Cycle | Carry (`y`) |
| ----- | ----------- |
| 1     | 0           |
| 2     | 1           |
| 3     | 1           |
| 4     | 1           |
| 5–8   | 1           |

Carry is propagated correctly throughout the operation.

---

## **7. Key Observations**

✔ One bit processed per cycle
✔ Correct carry propagation
✔ Output accumulates serially
✔ Operation stops after fixed cycles
✔ Final result is correct

<img width="1130" height="878" alt="image" src="https://github.com/user-attachments/assets/33e9433c-f09d-4c89-b761-becc71d5a257" />

<img width="1134" height="877" alt="image" src="https://github.com/user-attachments/assets/7897007b-085d-4a50-8404-09733d07c052" />

<img width="1676" height="922" alt="image" src="https://github.com/user-attachments/assets/5d28dce6-edba-4f2e-b7fc-971bfc6eded6" />



# **PART 5: VIVA, INTERVIEW, DEBUG & DESIGN INSIGHTS**

---

## **A) Viva Questions (University / Lab-Oriented)**

These questions focus on **basic understanding and explanation**, as typically expected in labs.

1. What is a serial adder?
2. Why is a serial adder slower than a parallel adder?
3. Why does a serial adder require memory?
4. What information is stored in the FSM of this design?
5. How many states are required for the FSM and why?
6. What does each FSM state represent?
7. Why is this serial adder implemented as a Mealy machine?
8. What is the role of the shift registers in this design?
9. Why is reset required in the serial adder?
10. How many clock cycles are required to add two 8-bit numbers?
11. What is the function of the counter in this design?
12. Why does the sum output change gradually over cycles?
13. What happens when the counter reaches zero?
14. How is carry propagated from one bit to the next?
15. What will happen if reset is not applied?

---

## **B) Interview-Style Questions (VLSI / Industry)**

These questions test **basic design awareness**, not deep research-level thinking.

1. What are the advantages of a serial adder over a parallel adder?
2. Where would you prefer to use a serial adder in real systems?
3. What is the main limitation of a serial adder?
4. How does FSM help in controlling the datapath?
5. What is the difference between Mealy and Moore FSMs?
6. Would this design work without a clock? Why?
7. How would you modify this design to indicate completion?
8. What happens if the input operands change during operation?
9. How would you scale this design for larger bit widths?
10. What blocks form the datapath and what blocks form the control?

---

## **C) Debug Scenarios (Realistic & Lab-Oriented)**

### **Bug 1: Using Blocking Assignment for State Register**

```verilog
always @(posedge clk)
    state = next_state;   // ❌ Wrong
```

**Problem:**

* Race conditions
* Simulation mismatch
* Incorrect FSM behavior

**Fix:**

```verilog
always @(posedge clk)
    state <= next_state;  // ✅ Correct
```

---

### **Bug 2: Missing Default Assignments in Combinational Logic**

```verilog
always @(*) begin
    case (state)
        ...
    endcase
end
```

**Problem:**

* Latch inference
* Unpredictable outputs

**Fix:**

```verilog
always @(*) begin
    next_state = state;
    s = 1'b0;
    ...
end
```

---

### **Bug 3: Output Logic Written in Sequential Block**

```verilog
always @(posedge clk)
    s <= QA[0] ^ QB[0];   // ❌ Wrong
```

**Problem:**

* Changes FSM behavior
* Introduces extra cycle delay

**Fix:**

```verilog
always @(*)
    s = QA[0] ^ QB[0];    // ✅ Correct
```

---

### **Bug 4: Incorrect Reset and Operand Loading Sequence**

```verilog
A = 8'h0B;
B = 8'h0D;
Reset = 0;   // ❌ Wrong
```

**Problem:**

* Operands not loaded into shift registers

**Fix:**

```verilog
Reset = 1;
A = 8'h0B;
B = 8'h0D;
#10 Reset = 0;   // ✅ Correct
```

---

### **Bug 5: Incorrect Shift Direction in Sum Register**

```verilog
Q[k] <= Q[k-1];   // ❌ Wrong
```

**Problem:**

* Reversed bit order
* Incorrect final sum

**Fix:**

```verilog
Q[k-1] <= Q[k];   // ✅ Correct
```

---

### **Bug 6: Counter Not Controlling the Operation**

```verilog
assign Run = 1'b1;   // ❌ Wrong
```

**Problem:**

* Extra shifts
* Corrupted output

**Fix:**

```verilog
assign Run = |Count; // ✅ Correct
```

---

## **D) Design Variations**

1. **Moore Serial Adder**

   * Output depends only on state
   * Sum appears one cycle later

2. **Parallel Adder**

   * Uses multiple full adders
   * Faster but higher area

3. **Parameterized Serial Adder**

   * Supports variable bit widths
   * Uses parameters instead of fixed sizes

4. **Handshake-Based Serial Adder**

   * Uses `valid` and `done` signals
   * Suitable for system-level designs

5. **Pipelined Serial Adder**

   * Improves throughput
   * Adds registers between stages

---

## **E) Conceptual Questions**

1. Why does serial addition trade speed for area?
2. How does FSM abstraction simplify sequential designs?
3. Why is carry treated as state information?
4. What makes this design sequential rather than combinational?
5. How does clock control correctness in this design?

---

## **F) If This Were in a Real Chip…**

In a practical implementation, additional aspects would be considered:

* Power optimization
* Clock quality and distribution
* Reset synchronization
* Data validity checks
* Verification using assertions
* Timing analysis and constraints

---

# **Part -6 : RTL Schematic**

---
<img width="1225" height="678" alt="image" src="https://github.com/user-attachments/assets/15a88959-e94c-45b3-8291-3167638c5d98" />

# **Part - 7: Constraints for implementation after eloboration**
'''
create_clock -period 10 -name Clock [get_ports Clock]

set_property PACKAGE_PIN E3 [get_ports Clock]
set_property IOSTANDARD LVCMOS33 [get_ports Clock]
set_property PACKAGE_PIN V10 [get_ports {A[7]}]
set_property PACKAGE_PIN U11 [get_ports {A[6]}]
set_property PACKAGE_PIN U12 [get_ports {A[5]}]
set_property PACKAGE_PIN H6 [get_ports {A[4]}]
set_property PACKAGE_PIN T13 [get_ports {A[3]}]
set_property PACKAGE_PIN R16 [get_ports {A[2]}]
set_property PACKAGE_PIN U8 [get_ports {A[1]}]
set_property PACKAGE_PIN T8 [get_ports {A[0]}]
set_property PACKAGE_PIN R13 [get_ports {B[7]}]
set_property PACKAGE_PIN U18 [get_ports {B[6]}]
set_property PACKAGE_PIN T18 [get_ports {B[5]}]
set_property PACKAGE_PIN R17 [get_ports {B[4]}]
set_property PACKAGE_PIN R15 [get_ports {B[3]}]
set_property PACKAGE_PIN M13 [get_ports {B[2]}]
set_property PACKAGE_PIN L16 [get_ports {B[1]}]
set_property PACKAGE_PIN J15 [get_ports {B[0]}]
set_property PACKAGE_PIN N17 [get_ports Reset]
set_property PACKAGE_PIN V11 [get_ports {Sum[7]}]
set_property PACKAGE_PIN V12 [get_ports {Sum[6]}]
set_property PACKAGE_PIN V14 [get_ports {Sum[5]}]
set_property PACKAGE_PIN V15 [get_ports {Sum[4]}]
set_property PACKAGE_PIN T16 [get_ports {Sum[3]}]
set_property PACKAGE_PIN U14 [get_ports {Sum[2]}]
set_property PACKAGE_PIN T15 [get_ports {Sum[1]}]
set_property PACKAGE_PIN V16 [get_ports {Sum[0]}]
set_property IOSTANDARD LVCMOS33 [get_ports Reset]
set_property IOSTANDARD LVCMOS33 [get_ports {A[7]}]
set_property IOSTANDARD LVCMOS33 [get_ports {A[6]}]
set_property IOSTANDARD LVCMOS33 [get_ports {A[5]}]
set_property IOSTANDARD LVCMOS33 [get_ports {A[4]}]
set_property IOSTANDARD LVCMOS33 [get_ports {A[3]}]
set_property IOSTANDARD LVCMOS33 [get_ports {A[2]}]
set_property IOSTANDARD LVCMOS33 [get_ports {A[1]}]
set_property IOSTANDARD LVCMOS33 [get_ports {A[0]}]
set_property IOSTANDARD LVCMOS33 [get_ports {B[7]}]
set_property IOSTANDARD LVCMOS33 [get_ports {B[6]}]
set_property IOSTANDARD LVCMOS33 [get_ports {B[5]}]
set_property IOSTANDARD LVCMOS33 [get_ports {B[4]}]
set_property IOSTANDARD LVCMOS33 [get_ports {B[3]}]
set_property IOSTANDARD LVCMOS33 [get_ports {B[2]}]
set_property IOSTANDARD LVCMOS33 [get_ports {B[1]}]
set_property IOSTANDARD LVCMOS33 [get_ports {B[0]}]
set_property IOSTANDARD LVCMOS33 [get_ports {Sum[7]}]
set_property IOSTANDARD LVCMOS33 [get_ports {Sum[6]}]
set_property IOSTANDARD LVCMOS33 [get_ports {Sum[5]}]
set_property IOSTANDARD LVCMOS33 [get_ports {Sum[4]}]
set_property IOSTANDARD LVCMOS33 [get_ports {Sum[2]}]
set_property IOSTANDARD LVCMOS33 [get_ports {Sum[3]}]
set_property IOSTANDARD LVCMOS33 [get_ports {Sum[1]}]
set_property IOSTANDARD LVCMOS33 [get_ports {Sum[0]}]
'''

# Part - 7: Final Summary

<img width="1640" height="854" alt="image" src="https://github.com/user-attachments/assets/18297268-6be2-406e-9cef-9e0bb6f0e85e" />

<img width="1640" height="853" alt="image" src="https://github.com/user-attachments/assets/ea7807f9-0f91-4016-867d-6dc61362cbad" />

<img width="835" height="180" alt="image" src="https://github.com/user-attachments/assets/ee773a44-a171-4f9a-b4a4-78dba895f54c" />


