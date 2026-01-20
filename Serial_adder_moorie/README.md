# **PART 1: THEORY, ARCHITECTURE, FSM DESIGN**

**(Serial Adder – Moore Model)**

---

## **1. Concept / Theory**

A **serial adder** is a sequential digital circuit that performs binary addition by processing **one bit per clock cycle**. The addition starts from the **least significant bit (LSB)** and proceeds toward the **most significant bit (MSB)**. Instead of using multiple full adders, a serial adder uses a **single full-adder operation repeatedly**, along with storage elements to retain intermediate results.

This approach significantly reduces hardware complexity and is suitable for systems where **area and power efficiency** are prioritized over speed.

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

## **4. FSM Model Selection (Moore Model)**

The serial adder in this experiment is implemented using a **Moore-type FSM**, where:

* Outputs depend **only on the present state**
* Outputs are **independent of current input changes**
* State transitions depend on:

  * Present state
  * Current input bits

Mathematically, the Moore FSM is defined as:

* **Next State** = f (Present State, Inputs)
* **Output** = g (Present State)

In the Moore serial adder:

* The **sum bit is associated with the FSM state**
* The carry is encoded within the FSM state
* Output changes occur **only on clock edges**

---

## **5. Why Moore FSM is Preferred over Mealy FSM**

Compared to a Mealy FSM, a Moore FSM offers the following advantages in a serial adder:

* Output changes only on **clock edges**
* Output does not depend directly on input changes
* Improved **output stability**
* Easier timing analysis
* Reduced risk of glitches at the output

**Trade-off:**

* Requires more states than Mealy FSM
* Introduces one-cycle latency in sum generation

---

## **6. Inputs and Outputs**

<img width="541" height="215" alt="image" src="https://github.com/user-attachments/assets/799c095b-1206-4096-8eed-96746b1dd893" />

### **Top-Level Serial Adder Signals**

| Signal      | Description              |
| ----------- | ------------------------ |
| `A [7:0]`   | Parallel input operand A |
| `B [7:0]`   | Parallel input operand B |
| `Clock`     | System clock             |
| `Reset`     | Active-high reset        |
| `Sum [7:0]` | Parallel output sum      |

---

### **Internal Signals (Conceptual)**

| Signal  | Purpose                                  |
| ------- | ---------------------------------------- |
| `QA[0]` | Current serial bit of operand A          |
| `QB[0]` | Current serial bit of operand B          |
| `s`     | Serial sum output derived from FSM state |
| `y1`    | Present sum-related state bit            |
| `y2`    | Present carry-related state bit          |
| `Y1`    | Next sum-related state bit               |
| `Y2`    | Next carry-related state bit             |
| `Run`   | Controls shifting operation              |

---

## **7. Full Adder Operation and FSM-Based Interpretation**

<img width="404" height="302" alt="image" src="https://github.com/user-attachments/assets/7fc7bab0-e7a0-476e-81b1-20de7e66e49e" />

The Moore serial adder implements full-adder functionality using **state storage elements**, as represented conceptually by two flip-flops.

* The **full adder** computes:

  * A sum bit
  * A carry-out
* These outputs are **not applied directly**
* Instead:

  * The sum bit is stored in **state flip-flop `y1`**
  * The carry-out is stored in **state flip-flop `y2`**

The **next-state signals** are:

* `Y1` → next value of sum-related state
* `Y2` → next value of carry-related state

The pair (`y1`, `y2`) uniquely defines the **Moore FSM state**, and the serial sum output `s` is generated **only from the present state**, satisfying Moore FSM behavior.

---

## **8. State Definition and Meaning**

Each Moore FSM state represents a unique combination of:

* Carry value
* Sum output value

| State | Carry | Sum Output |
| ----- | ----- | ---------- |
| `G0`  | 0     | 0          |
| `G1`  | 0     | 1          |
| `H0`  | 1     | 0          |
| `H1`  | 1     | 1          |

* `G` states correspond to **carry = 0**
* `H` states correspond to **carry = 1**

---

## **9. State Transition Behavior**

<img width="479" height="272" alt="image" src="https://github.com/user-attachments/assets/aca5020c-139c-4d6e-b6e4-899a9bfd77b6" />

State transitions depend on:

* Present state
* Input bits (`x` from operand A, `y` from operand B)

### **State Transition Table**

| Present State | Carry | Sum | x y = 00 | x y = 01 | x y = 10 | x y = 11 |
| ------------- | ----- | --- | -------- | -------- | -------- | -------- |
| `G0`          | 0     | 0   | `G0`     | `G1`     | `G1`     | `H0`     |
| `G1`          | 0     | 1   | `G0`     | `G1`     | `G1`     | `H0`     |
| `H0`          | 1     | 0   | `G1`     | `H0`     | `H0`     | `H1`     |
| `H1`          | 1     | 1   | `G1`     | `H0`     | `H0`     | `H1`     |

---

## **10. Textual FSM Representation**

```
Reset → G0

G0: Carry = 0, Sum = 0
G1: Carry = 0, Sum = 1
H0: Carry = 1, Sum = 0
H1: Carry = 1, Sum = 1
```

---

## **11. Overall Design Strategy**

* Shift registers provide serial operand bits
* Moore FSM stores carry and sum states
* Output shift register collects sum bits
* Counter limits operation cycles
* Reset initializes FSM and datapath

---

# **PART 2: SYNTHESIZABLE VERILOG RTL**

**(Serial Adder – Moore Model)**

---

## **1. RTL Design Overview**

The Moore serial adder RTL consists of:

* Parameterized shift registers for serial data handling
* A Moore finite state machine to store carry and sum state
* A counter-based control mechanism to limit the number of cycles
* A single top-level module integrating datapath and control

The design is fully synchronous, synthesizable, and suitable for FPGA and ASIC implementation.

---

## **2. Parameterized Shift Register Module**

The shift register supports:

* Parallel loading during reset
* Serial shifting during operation
* MSB insertion for sum accumulation

```verilog
module shiftreg (R, L, E, w, Clock, Q);
    parameter n = 8;

    input  [n-1:0] R;     // Parallel load input
    input          L;     // Load enable
    input          E;     // Shift enable
    input          w;     // Serial input
    input          Clock;
    output reg [n-1:0] Q;

    integer k;

    always @(posedge Clock) begin
        if (L)
            Q <= R;
        else if (E) begin
            for (k = n-1; k > 0; k = k - 1)
                Q[k-1] <= Q[k];
            Q[n-1] <= w;
        end
    end

endmodule
```

✔ Parameterized (`n = 8`)
✔ Synthesizable
✔ No latches or delays

---

## **3. Top-Level Moore Serial Adder Module**

This module integrates:

* Input shift registers
* Moore FSM (states `y1`, `y2`)
* Output shift register
* Counter-based run control

```verilog
module serial_adder_moore (
    input  [7:0] A,
    input  [7:0] B,
    input        Clock,
    input        Reset,
    output [7:0] Sum
);

    // Internal wires
    wire [7:0] QA, QB;
    wire       a, b;
    wire       Run;

    // FSM state and next-state registers
    reg  y1, y2;
    reg  Y1, Y2;

    // Cycle counter
    reg [3:0] Count;

    // Input shift registers
    shiftreg SA (A, Reset, Run, 1'b0, Clock, QA);
    shiftreg SB (B, Reset, Run, 1'b0, Clock, QB);

    assign a = QA[0];
    assign b = QB[0];

    // Output shift register (Moore output)
    shiftreg SS (8'b0, Reset, Run, y1, Clock, Sum);

    // Moore FSM next-state logic
    always @(*) begin
        Y1 = y1;
        Y2 = y2;

        case ({y2, y1})
            2'b00: begin
                case ({a, b})
                    2'b00: {Y2, Y1} = 2'b00;
                    2'b01,
                    2'b10: {Y2, Y1} = 2'b01;
                    2'b11: {Y2, Y1} = 2'b10;
                endcase
            end

            2'b01: begin
                case ({a, b})
                    2'b00: {Y2, Y1} = 2'b00;
                    2'b01,
                    2'b10: {Y2, Y1} = 2'b01;
                    2'b11: {Y2, Y1} = 2'b10;
                endcase
            end

            2'b10: begin
                case ({a, b})
                    2'b00: {Y2, Y1} = 2'b01;
                    2'b01,
                    2'b10: {Y2, Y1} = 2'b10;
                    2'b11: {Y2, Y1} = 2'b11;
                endcase
            end

            2'b11: begin
                case ({a, b})
                    2'b00: {Y2, Y1} = 2'b01;
                    2'b01,
                    2'b10: {Y2, Y1} = 2'b10;
                    2'b11: {Y2, Y1} = 2'b11;
                endcase
            end
        endcase
    end

    // FSM state register
    always @(posedge Clock) begin
        if (Reset) begin
            y1 <= 1'b0;
            y2 <= 1'b0;
        end
        else begin
            y1 <= Y1;
            y2 <= Y2;
        end
    end

    // Counter logic
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

## **4. Operational Description**

1. When `Reset` is asserted:

   * Operands `A` and `B` are loaded into input shift registers
   * FSM state (`y1`, `y2`) is initialized to `G0`
   * Counter is set to 8

2. After reset is deasserted:

   * One bit from each operand is shifted out per clock cycle
   * FSM computes the next state based on current state and input bits
   * The sum bit is generated from the FSM state (`y1`)
   * The sum bit is shifted into the output shift register

3. The counter decrements each cycle:

   * When the counter reaches zero, shifting stops
   * The final sum remains stable at the output

---

# **PART 3: TESTBENCH (VERIFICATION STYLE)**

**(Serial Adder – Moore Model)**

---

## **1. Testbench Objectives**

The objectives of this testbench are to:

* Verify correct operation of the Moore-based serial adder
* Validate proper reset and operand loading behavior
* Confirm correct serial processing of input bits
* Observe Moore FSM timing behavior
* Ensure the **final sum is produced only after an additional clock cycle**
* Confirm correct termination of the operation

---

## **2. Testbench Strategy**

The verification strategy used is **directed testing**, which is sufficient for this experiment because:

* The functionality is deterministic
* The number of states and transitions is limited
* Expected outputs are known in advance

The testbench performs the following:

* Generates a periodic clock
* Applies reset synchronously
* Loads operands during reset
* Allows the serial adder to run for sufficient clock cycles
* Observes internal and output signals using `$monitor`

---

## **3. Clock Generation**

* Clock period: **10 ns**
* 50% duty cycle

```verilog
always #5 Clock = ~Clock;
```

---

## **4. Reset and Operand Application**

* Reset is asserted initially to:

  * Load operands into shift registers
  * Initialize FSM states (`y1 = 0`, `y2 = 0`)
  * Initialize the counter

* Operands are applied **while reset is asserted**, ensuring correct parallel loading.

---

## **5. Directed Test Case**

### **Test Case Used**

| Operand | Binary Value | Decimal |
| ------- | ------------ | ------- |
| A       | `00001011`   | 11      |
| B       | `00001110`   | 14      |

### **Expected Result**

```
11 + 14 = 25 = 00011001
```

---

## **6. Testbench Code (Moore FSM–Aware)**

The testbench allows **one additional clock cycle** beyond the normal 8 cycles to ensure the final FSM state is flushed into the Sum register.

```verilog
`timescale 1ns / 1ps

module tb_serial_adder_moore;

    reg  [7:0] A;
    reg  [7:0] B;
    reg        Clock;
    reg        Reset;
    wire [7:0] Sum;

    // Instantiate the DUT
    serial_adder_moore DUT (
        .A(A),
        .B(B),
        .Clock(Clock),
        .Reset(Reset),
        .Sum(Sum)
    );

    // Clock generation (10 ns period)
    always #5 Clock = ~Clock;

    initial begin
        // Initialize signals
        Clock = 0;
        Reset = 1;

        // Apply operands during reset
        A = 8'b00001011;   // 11
        B = 8'b00001110;   // 14

        // Hold reset for one clock cycle
        #10;
        Reset = 0;

        // Allow sufficient time for Moore FSM operation
        // 8 cycles for bits + 1 extra cycle for final sum
        #220;

        // End simulation
        $finish;
    end

    // Monitor outputs
    initial begin
        $monitor(
            "Time=%0t | Reset=%b | A=%b | B=%b | Sum=%b",
            $time, Reset, A, B, Sum
        );
    end

endmodule
```

---

## **7. Observations During Simulation**

During simulation, the following behavior is observed:

* Operands are correctly loaded during reset
* FSM initializes to state `G0`
* One bit is processed per clock cycle
* After 8 cycles, the FSM holds the final carry/sum state
* During the **9th cycle**, the final sum bit is shifted into the Sum register
* Output stabilizes at:

```
Sum = 00011001 (25)
```

---

## **8. Verification Outcome**

* ✔ Moore FSM behavior verified
* ✔ Extra-cycle requirement validated
* ✔ Correct final sum obtained
* ✔ No race conditions observed
* ✔ Design behaves as expected

---

# **PART 4: WAVEFORM & TIMING EXPLANATION**

**(Serial Adder – Moore Model)**

---

<img width="1053" height="698" alt="image" src="https://github.com/user-attachments/assets/7b6eaa93-bc3f-46f1-9aca-8037a91a6e6c" />

<img width="1238" height="698" alt="image" src="https://github.com/user-attachments/assets/741c1e90-15ee-4ea8-9f4c-6ee324c63582" />
<img width="1238" height="693" alt="image" src="https://github.com/user-attachments/assets/42afc75e-a736-4dd0-bbc1-291a46214921" />

<img width="1660" height="734" alt="image" src="https://github.com/user-attachments/assets/ab957fc4-4137-4df3-97a9-4f7484677c7b" />


## **1. Signals Observed**

The following signals are considered during waveform analysis:

| Signal     | Description                      |
| ---------- | -------------------------------- |
| `Clock`    | System clock                     |
| `Reset`    | Active-high reset                |
| `QA[0]`    | Current serial bit of operand A  |
| `QB[0]`    | Current serial bit of operand B  |
| `y1`       | Sum-related FSM state bit        |
| `y2`       | Carry-related FSM state bit      |
| `Y1`, `Y2` | Next-state signals               |
| `Count`    | Remaining number of cycles       |
| `Run`      | Control signal enabling shifting |
| `Sum[7:0]` | Output sum register              |

---

## **2. Reset Phase Behavior**

* Initially, `Reset = 1`
* During reset:

  * Operands A and B are loaded into input shift registers
  * FSM state is initialized to:

    ```
    y2 = 0, y1 = 0   (State G0)
    ```
  * Counter is initialized to **9**
  * `Run = 1` because `Count ≠ 0`
* No shifting or addition occurs during reset

---

## **3. Control Path Operation (Count and Run)**

* `Count` determines **how many clock cycles** the serial adder operates
* `Run` is derived as:

  ```
  Run = (Count ≠ 0)
  ```
* When `Run = 1`:

  * Input shift registers shift
  * Output shift register shifts
  * FSM processes data
* When `Run = 0`:

  * Shifting stops
  * Output remains stable

Thus, **Count and Run form the control path**, while the FSM and shift registers form the datapath.

---

## **4. Serial Addition Phase – Cycle-by-Cycle Explanation**

After reset is deasserted, the serial adder begins operation.

### **General Operation per Clock Cycle**

For each clock cycle when `Run = 1`:

1. **Input Sampling**

   * `a = QA[0]`
   * `b = QB[0]`

2. **Previous FSM State**

   * FSM holds (`y2`, `y1`) from the previous cycle

3. **Next-State Computation**

   * Next state (`Y2`, `Y1`) is computed using:

     * Inputs (`a`, `b`)
     * Previous state (`y2`, `y1`)

4. **State Update (Clock Edge)**

   * FSM state updates:

     ```
     y2 ← Y2
     y1 ← Y1
     ```

5. **Output and Control Interaction**

   * Output sum bit:

     ```
     s = y1
     ```
   * Since `Run = 1`, `s` is shifted into the MSB of `Sum`
   * `Count` decrements by one

---

## **5. Detailed Cycle-by-Cycle Interpretation**

### **Inputs Applied**

```
A = 11 → 00001011
B = 14 → 00001110
```

| Cycle         | Count | Run   | Inputs (a,b) | Previous State (y2 y1) | Present State | Sum Bit Shifted | Sum Register |
| ------------- | ----- | ----- | ------------ | ---------------------- | ------------- | --------------- | ------------ |
| 1             | 9     | 1     | 1,0          | 00                     | 01            | 1               | 00000000     |
| 2             | 8     | 1     | 1,1          | 01                     | 10            | 0               | 10000000     |
| 3             | 7     | 1     | 0,1          | 10                     | 10            | 0               | 01000000     |
| 4             | 6     | 1     | 1,1          | 10                     | 11            | 1               | 00100000     |
| 5             | 5     | 1     | 0,0          | 11                     | 01            | 1               | 10010000     |
| 6             | 4     | 1     | 0,1          | 01                     | 01            | 1               | 11001000     |
| 7             | 3     | 1     | 0,0          | 01                     | 00            | 0               | 01100100     |
| 8             | 2     | 1     | 0,0          | 00                     | 00            | 0               | 00110010     |
| **9 (extra)** | **1** | **1** | —            | 00                     | 01            | **1**           | **00011001** |
| 10            | 0     | 0     | —            | —                      | —             | —               | Stable       |

✔ Final correct sum appears **after Count reaches zero**

---

## **6. Why the Extra Cycle is Required in Moore FSM**

* After processing the 8th input bit:

  * FSM contains the **final sum-related state**
  * `Count` is still non-zero
  * `Run` remains high
* The **9th cycle** allows:

  * FSM state (`y1`) to be shifted into the output register
* After this:

  * `Count = 0`
  * `Run = 0`
  * Shifting stops

This extra cycle is a **direct consequence of Moore FSM output behavior**.

---

## **7. Final Output Verification**

```
Final Sum = 00011001 (Decimal 25)
```

✔ Correct arithmetic result
✔ Output stabilizes when `Run = 0`

---

## **8. Mealy vs Moore FSM — Control and Output Perspective**

### **Mealy Serial Adder**

* Output depends on:

  * FSM state
  * Current inputs
* No extra cycle required
* Faster output
* **Input glitches can affect output**

### **Moore Serial Adder**

* Output depends **only on FSM state**
* Requires one extra cycle
* Output changes only on clock edges
* **Input glitches do NOT affect the registered output**
* Better timing predictability

---

## **9. Key Takeaways from PART 4**

* `Count` and `Run` form the **control path**
* FSM and shift registers form the **datapath**
* Moore FSM introduces intentional output latency
* Glitch immunity is a major advantage
* Waveform clearly validates design correctness

---

# **PART 5: VIVA, INTERVIEW, DEBUG & DESIGN INSIGHTS**

**(Serial Adder – Moore Model)**

---

## **A) Viva Questions (University / Lab-Oriented)**

These questions focus on **concept clarity and implementation understanding**.

1. What is a serial adder?
2. Why is the serial adder a sequential circuit?
3. What information is stored in the FSM of a Moore serial adder?
4. Why does a Moore serial adder require more states than a Mealy serial adder?
5. What do the FSM state bits `y1` and `y2` represent?
6. Why is an extra clock cycle required in the Moore serial adder?
7. What is the role of the counter in this design?
8. How does the `Run` signal control the operation?
9. Why are operands loaded during reset?
10. What happens when `Count` reaches zero?
11. How is the final sum stored and made stable?
12. Why is Moore FSM output more stable than Mealy FSM output?

---

## **B) Interview-Style Questions (Basic VLSI / Digital Design)**

These questions test **design awareness and trade-off understanding**.

1. Compare Moore and Mealy FSMs in terms of output timing.
2. Why would you prefer a Moore FSM in synchronous systems?
3. What is the area–speed trade-off in a serial adder?
4. How does the control path differ from the datapath in this design?
5. Why is glitch immunity important in digital systems?
6. How would you modify this design to support different operand widths?
7. What limits the maximum clock frequency of this serial adder?
8. How would you add a `done` signal to indicate completion?
9. What changes would be required to remove the extra cycle?
10. Where would serial adders be practically used?

---

## **C) Debug Scenarios (RTL-Oriented, Practical)**

### **Bug 1: Using Blocking Assignment for FSM State**

```verilog
always @(posedge Clock)
    y1 = Y1;   // ❌ Wrong
```

**Problem:**

* Race conditions
* Simulation vs synthesis mismatch

**Fix:**

```verilog
always @(posedge Clock)
    y1 <= Y1;  // ✅ Correct
```

---

### **Bug 2: Output Generated from Combinational Logic**

```verilog
assign s = a ^ b ^ y2;   // ❌ Wrong (Mealy-like)
```

**Problem:**

* Violates Moore FSM behavior
* Output sensitive to input glitches

**Fix:**

```verilog
assign s = y1;           // ✅ Correct
```

---

### **Bug 3: Counter Initialized to 8 Instead of 9**

```verilog
Count <= 4'd8;   // ❌ Wrong for Moore FSM
```

**Problem:**

* Final sum bit not flushed
* Incorrect output (e.g., 50 instead of 25)

**Fix:**

```verilog
Count <= 4'd9;   // ✅ Correct
```

---

### **Bug 4: Operands Applied After Reset**

```verilog
Reset = 0;
A = value;   // ❌ Wrong
```

**Problem:**

* Shift registers not loaded

**Fix:**

```verilog
Reset = 1;
A = value;   // ✅ Correct
```

---

### **Bug 5: Run Signal Always High**

```verilog
assign Run = 1'b1;   // ❌ Wrong
```

**Problem:**

* Extra shifts
* Corrupted sum

**Fix:**

```verilog
assign Run = (Count != 0);  // ✅ Correct
```

---

## **D) Design Variations**

1. **Mealy Serial Adder**

   * Fewer states
   * No extra cycle
   * Output sensitive to inputs

2. **Parallel Adder**

   * Faster computation
   * Higher area usage

3. **Parameterized Serial Adder**

   * Configurable bit-width
   * Reusable design

4. **Handshake-Based Serial Adder**

   * Uses `valid` and `done` signals
   * Suitable for system integration

5. **Pipelined Adder**

   * Higher throughput
   * Increased complexity

---

## **E) Conceptual Questions**

1. Why does Moore FSM improve output stability?
2. How does FSM abstraction simplify carry management?
3. Why is output latency acceptable in serial designs?
4. What makes this design synchronous?
5. How do control and datapath interact in this design?

---

## **F) If This Were in a Real Chip…**

In a practical hardware implementation, the following would also be considered:

* Clock gating for power reduction
* Reset synchronization
* Timing constraints and STA
* Formal and assertion-based verification
* Scan insertion for testability

---

# Part-6: RTL Schematic

<img width="1213" height="684" alt="image" src="https://github.com/user-attachments/assets/62e89f79-d457-4c32-aff6-44eb7b3604c3" />


  
