
# **Three-Way Traffic Light Controller – T-Junction**

---

# **PART 1: THEORY, ARCHITECTURE, FSM DESIGN**

---

## **1. Concept / Theory**

A **three-way traffic light controller** is a sequential digital system used to regulate traffic at a **T-junction**, where two main roads form a continuous straight path and a third road joins as a side road.

The controller ensures:

* Safe and orderly traffic flow
* No conflicting green signals
* Proper warning transitions using yellow lights
* Deterministic time-based sequencing

The system operates continuously under clock control using a finite state machine.

---

## **2. Need for Sequential Operation**

Traffic control is inherently **time-dependent** and requires memory of past activity.

Sequential operation is required because:

* Each traffic phase must last for a **fixed duration**
* Transitions depend on **elapsed time**, not only inputs
* The controller must remember the **current traffic phase**
* Outputs must change **synchronously with the clock**

Hence, a clocked sequential design is essential.

---

## **3. Role of Finite State Machine (FSM)**

The FSM provides structured control over traffic sequencing.

It:

* Represents each traffic phase as a distinct state
* Controls which lights are active in each phase
* Enables timers for phase duration
* Uses timer completion to trigger state transitions
* Handles special operating modes such as blink mode

The FSM guarantees safe, predictable operation.

---

## **4. Three-Way T-Junction Scenario and Modes of Operation**

### **Junction Description**

The assumed junction consists of:

* **Two main roads** forming a straight-through path
* **One side road** terminating at the junction

Main roads operate together, while the side road is served only when main roads are stopped.

---

### **Modes of Operation**

The controller supports **two operating modes**:

#### **Normal Traffic Mode**

* Main roads and side road operate in a fixed sequence
* Green, yellow, and red phases occur with predefined timing
* Traffic flow alternates safely between main and side roads

#### **Blink Mode**

* Activated during non-traffic hours
* All yellow lights flash simultaneously
* Overrides normal traffic sequencing
* Has the highest priority in the system

---

## **5. Traffic Phases and Timing**

| Phase       | Description                | Duration |
| ----------- | -------------------------- | -------- |
| Main Green  | Main roads allowed to move | 45 s     |
| Main Yellow | Transition warning         | 5 s      |
| Side Green  | Side road allowed to move  | 25 s     |
| Side Yellow | Transition warning         | 5 s      |
| Blink       | All yellow flashing        | 0.5 s    |

---

## **6. Datapath Architecture**

The datapath is responsible for **all timing-related operations**.
It converts the high-frequency system clock into **human-perceivable time intervals**.

---

### **6.1 Generation of 0.1 s Time Base from 50 MHz Clock**

The controller operates with a **50 MHz system clock**, which corresponds to:

[
\text{Clock period} = 20,\text{ns}
]

Traffic signals require durations in seconds.
Therefore, a **0.1 s time base** is generated using a free-running counter.

---

### **6.2 Free-Running Base Counter Operation**

* A 22-bit counter (`cnt1_reg`) increments on every rising clock edge
* When the counter reaches **4,999,999**, it resets to zero

This corresponds to:

[
4{,}999{,}999 \times 20,\text{ns} = 0.1,\text{s}
]

Thus, the counter produces a **0.1 s timing tick**, which forms the fundamental time unit for the controller.

This counter runs **continuously**, independent of FSM state.

---

### **6.3 Derived Timers**

Using the 0.1 s time base, multiple derived timers are implemented:

| Timer   | Purpose                  |
| ------- | ------------------------ |
| Timer-1 | Main road green duration |
| Timer-2 | Yellow phases            |
| Timer-3 | Side road green duration |
| Timer-4 | Blink mode               |

Each timer counts **0.1 s ticks**, not raw clock cycles.

---

### **6.4 Role of `cnt*_next` Signals**

For each timer counter, a corresponding **next-value signal** (`cnt*_next`) computes:

[
\text{Next Count} = \text{Current Count} + 1
]

These signals:

* Separate combinational next-value logic from sequential storage
* Ensure synchronous counter updates
* Prevent latch inference

The FSM controls **when** these next values are loaded.

---

## **7. Control Path Architecture**

The control path is implemented entirely using the FSM and governs **when timers operate and when states change**.

---

### **7.1 Control Path Responsibilities**

* Enable specific timers
* Monitor timer completion
* Control FSM state transitions
* Override normal operation during blink mode

---

### **7.2 Timer Enable Logic**

Each timer advances only when enabled by the FSM.

This condition is expressed logically as:

> **Timer Advance Condition**
>
> **start_timer_* = 1 AND (cnt1_reg = time_base)**

This ensures that timers increment **only once per 0.1 s interval**.

---

### **7.3 Timer Completion Logic**

A timer completes when:

> **(cnt1_reg = time_base) AND (timer_count = terminal_value)**

This generates a completion flag used by the FSM.

---

### **7.4 Timer Completion Flags**

| Flag       | Meaning                  |
| ---------- | ------------------------ |
| `res_cnt2` | Main green completed     |
| `res_cnt3` | Yellow completed         |
| `res_cnt4` | Side green completed     |
| `res_cnt5` | Blink interval completed |

These flags form the interface between datapath and control path.

---

### **7.5 Blink Mode Priority Rule**

Blink mode has **highest priority**:

> If `blink = 1`, the FSM transitions immediately to blink state, regardless of current state.

Normal operation resumes when `blink` is deasserted.

---

## **8. FSM Design**

### **8.1 FSM State Definitions**

| State | Description                      |
| ----- | -------------------------------- |
| S0    | Main roads green, side road red  |
| S1    | Main roads yellow, side road red |
| S2    | Main roads red, side road green  |
| S3    | Main roads red, side road yellow |
| S4    | Blink mode                       |

---

### **8.2 FSM State Transitions**

```
S0 → S1 → S2 → S3 → S0
```

Blink mode overrides all transitions.

---

### **8.3 FSM Type Justification**

A **Moore FSM** is used because:

* Outputs depend only on the current state
* Outputs are stable and glitch-free
* Safety is prioritized over latency

---

## **9. Inputs and Outputs**

### **9.1 Top-Level Input Signals**

| Signal    | Description                       |
| --------- | --------------------------------- |
| `clk`     | System clock (50 MHz in hardware) |
| `reset_n` | Active-low asynchronous reset     |
| `blink`   | Enables blink mode operation      |

---

### **9.2 Traffic Light Output Signals**

#### **Main Road Signals**

| Signal       | Description             |
| ------------ | ----------------------- |
| `MG1`, `MG2` | Main road green lights  |
| `MY1`, `MY2` | Main road yellow lights |
| `MR1`, `MR2` | Main road red lights    |

#### **Side Road Signals**

| Signal | Description            |
| ------ | ---------------------- |
| `SG`   | Side road green light  |
| `SY`   | Side road yellow light |
| `SR`   | Side road red light    |

*(Main road signals operate in pairs, reflecting the straight-through main road.)*

---

### **9.3 Internal Control and Timing Signals (Conceptual)**

| Signal          | Purpose                       |
| --------------- | ----------------------------- |
| `state`         | FSM present state             |
| `start_timer_1` | Enables main road green timer |
| `start_timer_2` | Enables yellow timer          |
| `start_timer_3` | Enables side road green timer |
| `cnt1_reg`      | Base time counter (0.1 s)     |
| `cnt2_reg`      | Main green timer              |
| `cnt3_reg`      | Yellow timer                  |
| `cnt4_reg`      | Side green timer              |
| `cnt5_reg`      | Blink timer                   |
| `res_cnt*`      | Timer completion flags        |

---

## **10. Timing Considerations for Simulation**

### **10.1 Hardware Timing Perspective**

In real hardware:

* Clock frequency = **50 MHz**
* Clock period = **20 ns**
* Base time unit = **0.1 s**, generated using a 22-bit counter
* 
Simulating these durations directly would require **millions of clock cycles**, making simulation slow and resource-intensive.

---

### **10.2 Simulation Time Scaling**

To make simulation practical:

* The **same RTL design** is retained
* The **clock period is scaled down** in the testbench
* The base time (`0.1 s`) is interpreted as a **much smaller simulation time unit**

* **Hardware base time:** 0.1 s
* **Simulation base time:** 0.1 ns

The internal counter structure and FSM logic remain unchanged.
Only the **clock period in the testbench** is reduced.

---

### **10.3 Real-Time vs Simulation-Time Mapping**

| Phase              | Real Time | Simulation Time |
| ------------------ | --------- | --------------- |
| Base time          | 0.1 s     | 0.1 ns          |
| Main road straight | 45 s      | 45 ns           |
| Side road          | 25 s      | 25 ns           |
| Yellow phase       | 5 s       | 5 ns            |
| Blink mode         | 0.5 s     | 0.5 ns          |

---

### **10.4 Effect of Time Scaling**

* FSM sequencing remains identical
* Timer enable and completion logic remain unchanged
* Relative timing between phases is preserved
* Only the absolute duration is compressed

> **This scaling affects only simulation, not hardware behavior.**

---

### **10.5 Verification Benefit**

* Faster simulation execution
* Complete traffic cycle observable in waveform
* Functional correctness verified without modifying RTL

This approach is a **standard industry practice** for verifying long-duration control systems.

---

## **11. Design Strategy and Rationale**

* Moore FSM is used to ensure **glitch-free and stable traffic light outputs**
* Counter-based timing provides **accurate and deterministic phase durations**
* A shared base counter efficiently generates all required timing intervals
* Simplified FSM structure reduces design complexity for a three-way junction
* Conservative use of yellow phases improves **traffic safety**
* Priority-based blink mode handling ensures **robust operation during special conditions**

---

## **12. Summary**

* The three-way traffic light controller is a **time-driven, FSM-controlled system**
* Datapath performs timing generation using counters
* Control path sequences traffic phases using FSM logic
* Blink mode overrides normal operation for safety
* The design is **fully synthesizable**, simulation-friendly, and suitable for **FPGA and real-world traffic control systems**

---

# **PART 2: SYNTHESIZABLE RTL DESIGN**

---

## **2.1 Design Requirements**

The three-way traffic light controller must satisfy the following requirements:

* Control a **T-junction** with:

  * Two main roads (straight-through traffic)
  * One side road
* Support **normal traffic mode** and **blink (night/emergency) mode**
* Generate **human-perceivable timing** from a high-frequency system clock
* Ensure **safe, non-conflicting traffic flow**
* Be fully **synthesizable** and **tool-verifiable**
* Maintain **textbook-consistent architecture**, reused from the four-way controller

---

## **2.2 Overall RTL Architecture**

The RTL design follows a **clean separation of responsibilities**:

| Block        | Responsibility                              |
| ------------ | ------------------------------------------- |
| Datapath     | Time-base generation and timing measurement |
| Control Path | FSM-based sequencing and decision making    |
| Output Logic | Driving traffic lights safely               |

The controller is implemented as a **Moore FSM**, ensuring that all traffic light outputs change only on clock edges.

---

## **2.3 Module Interface Definition**

### **Top-Level Input Signals**

| Signal    | Purpose                              |
| --------- | ------------------------------------ |
| `clk`     | System clock (50 MHz in hardware)    |
| `reset_n` | Active-low asynchronous reset        |
| `blink`   | Enables blink (night/emergency) mode |

---

### **Traffic Light Output Signals – Main Roads**

| Signal       | Purpose                 |
| ------------ | ----------------------- |
| `MG1`, `MG2` | Main road green lights  |
| `MY1`, `MY2` | Main road yellow lights |
| `MR1`, `MR2` | Main road red lights    |

---

### **Traffic Light Output Signals – Side Road**

| Signal | Purpose                |
| ------ | ---------------------- |
| `SG`   | Side road green light  |
| `SY`   | Side road yellow light |
| `SR`   | Side road red light    |

---

## **2.4 Internal Timing Signals and Counters**

### **Counter Registers**

| Signal           | Purpose                                             |
| ---------------- | --------------------------------------------------- |
| `cnt1_reg[21:0]` | Free-running base counter providing 0.1 s time base |
| `cnt2_reg[8:0]`  | Main road green timer (45 s)                        |
| `cnt3_reg[5:0]`  | Yellow phase timer (5 s)                            |
| `cnt4_reg[7:0]`  | Side road green timer (25 s)                        |
| `cnt5_reg[7:0]`  | Blink mode timer (0.5 s)                            |

---

### **Next-Value Signals (`cnt*_next`)**

The `cnt*_next` signals represent the **incremented value** of each counter before it is registered.

**Purpose:**

* Ensure clean synchronous design
* Make counter behavior explicit
* Avoid implicit arithmetic in sequential blocks
* Improve synthesis clarity and readability

Each counter:

* Increments only when enabled
* Resets when its terminal count is reached
* Otherwise retains its value

---

## **2.5 Datapath Timing Strategy**

### **Clock to Time-Base Conversion**

* System clock: **50 MHz**
* Clock period: **20 ns**

To obtain human-scale timing, the datapath generates a **0.1 s base time** using a free-running counter.

**Concept:**

* Count 4,999,999 clock cycles
* Reset counter after reaching this value
* Generate a periodic **0.1 s timing tick**

This tick drives all higher-level traffic timing.

---

### **Derived Timers**

Using the 0.1 s base tick, the datapath implements:

| Timer   | Purpose            | Duration |
| ------- | ------------------ | -------- |
| Timer-1 | Main road straight | 45 s     |
| Timer-2 | Yellow transition  | 5 s      |
| Timer-3 | Side road traffic  | 25 s     |
| Timer-4 | Blink mode         | 0.5 s    |

---

# **2.6: RTL Code**

---

```verilog
`timescale 1ns / 1ps
/*---------------------------------------------------------
  Three-Way Traffic Light Controller (T-Junction)
  Derived directly from textbook 4-way controller
---------------------------------------------------------*/

/*---------------- State Definitions ----------------*/
`define S0   4'd0   // Main green
`define S1   4'd1   // Main yellow
`define S2   4'd2   // Side green
`define S3   4'd3   // Side yellow
`define S12  4'd12  // Blink mode

/*---------------- Timing Parameters ----------------*/
`define time_base 23'd4999999
/* for simulation we down scale the base timer to 0.1ns, thus time base needs just upto count of 4
/* `define time_base 23'b4 */ //only for simulation purpose
`define load_cnt2 9'd449   // 45 s
`define load_cnt3 6'd49    // 5 s
`define load_cnt4 8'd249   // 25 s
`define load_cnt5 8'd4     // 0.5 s

/*--------------------------------------------------*/
module traffic_controller_3way (
    input  clk,
    input  reset_n,
    input  blink,

    /* Main road signals */
    output reg MR1, MR2,
    output reg MY1, MY2,
    output reg MG1, MG2,

    /* Side road signals */
    output reg SR,
    output reg SY,
    output reg SG
);

/*---------------- Internal Wires ----------------*/
wire adv_cnt2, adv_cnt3, adv_cnt4, adv_cnt5;
wire res_cnt2, res_cnt3, res_cnt4, res_cnt5;

/*---------------- Counters ----------------*/
reg  [23:0] cnt1_reg;
reg  [8:0]  cnt2_reg;
reg  [5:0]  cnt3_reg;
reg  [7:0]  cnt4_reg;
reg  [7:0]  cnt5_reg;

/*---------------- Next Counters ----------------*/
wire [23:0] cnt1_next = cnt1_reg + 1;
wire [8:0]  cnt2_next = cnt2_reg + 1;
wire [5:0]  cnt3_next = cnt3_reg + 1;
wire [7:0]  cnt4_next = cnt4_reg + 1;
wire [7:0]  cnt5_next = cnt5_reg + 1;

/*---------------- Timer Control ----------------*/
reg start_timer_1;
reg start_timer_2;
reg start_timer_3;

/*---------------- FSM State ----------------*/
reg [3:0] state;

/*--------------------------------------------------
  Free-Running Base Counter (0.1s)
--------------------------------------------------*/
always @(posedge clk or negedge reset_n) begin
    if (!reset_n)
        cnt1_reg <= 22'd0;
    else if (cnt1_reg == `time_base)
        cnt1_reg <= 22'd0;
    else
        cnt1_reg <= cnt1_next;
end

/* Timer enables */
assign adv_cnt2 = start_timer_1 & (cnt1_reg == `time_base);
assign adv_cnt3 = start_timer_2 & (cnt1_reg == `time_base);
assign adv_cnt4 = start_timer_3 & (cnt1_reg == `time_base);
assign adv_cnt5 = blink         & (cnt1_reg == `time_base);

/* Timer completions */
assign res_cnt2 = adv_cnt2 & (cnt2_reg == `load_cnt2);
assign res_cnt3 = adv_cnt3 & (cnt3_reg == `load_cnt3);
assign res_cnt4 = adv_cnt4 & (cnt4_reg == `load_cnt4);
assign res_cnt5 = adv_cnt5 & (cnt5_reg == `load_cnt5);

/* Timer registers */
always @(posedge clk or negedge reset_n) begin
    if (!reset_n) cnt2_reg <= 0;
    else if (res_cnt2) cnt2_reg <= 0;
    else if (adv_cnt2) cnt2_reg <= cnt2_next;
end

always @(posedge clk or negedge reset_n) begin
    if (!reset_n) cnt3_reg <= 0;
    else if (res_cnt3) cnt3_reg <= 0;
    else if (adv_cnt3) cnt3_reg <= cnt3_next;
end

always @(posedge clk or negedge reset_n) begin
    if (!reset_n) cnt4_reg <= 0;
    else if (res_cnt4) cnt4_reg <= 0;
    else if (adv_cnt4) cnt4_reg <= cnt4_next;
end

always @(posedge clk or negedge reset_n) begin
    if (!reset_n) cnt5_reg <= 0;
    else if (res_cnt5) cnt5_reg <= 0;
    else if (adv_cnt5) cnt5_reg <= cnt5_next;
end

/*--------------------------------------------------
  FSM and Output Logic (Moore)
--------------------------------------------------*/
always @(posedge clk or negedge reset_n) begin
    if (!reset_n) begin
        MR1 <= 0; MR2 <= 0;
        MY1 <= 0; MY2 <= 0;
        MG1 <= 0; MG2 <= 0;
        SR  <= 0; SY  <= 0; SG <= 0;
        start_timer_1 <= 0;
        start_timer_2 <= 0;
        start_timer_3 <= 0;
        state <= `S0;
    end
    else begin

        MR1 <= 0; MR2 <= 0;
        MY1 <= 0; MY2 <= 0;
        MG1 <= 0; MG2 <= 0;
        SR  <= 0; SY  <= 0; SG <= 0;

        case (state)

        `S0: begin
            if (blink) state <= `S12;
            else begin
                MG1 <= 1; MG2 <= 1;
                SR  <= 1;
                start_timer_1 <= ~res_cnt2;
                if (res_cnt2) state <= `S1;
            end
        end

        `S1: begin
            if (blink) state <= `S12;
            else begin
                MY1 <= 1; MY2 <= 1;
                SR  <= 1;
                start_timer_2 <= ~res_cnt3;
                if (res_cnt3) state <= `S2;
            end
        end

        `S2: begin
            if (blink) state <= `S12;
            else begin
                MR1 <= 1; MR2 <= 1;
                SG  <= 1;
                start_timer_3 <= ~res_cnt4;
                if (res_cnt4) state <= `S3;
            end
        end

        `S3: begin
            if (blink) state <= `S12;
            else begin
                MR1 <= 1; MR2 <= 1;
                SY  <= 1;
                start_timer_2 <= ~res_cnt3;
                if (res_cnt3) state <= `S0;
            end
        end

        `S12: begin
            if (res_cnt5) begin
                MY1 <= ~MY1;
                MY2 <= ~MY2;
                SY  <= ~SY;
            end
            if (!blink) state <= `S0;
        end

        default: state <= `S0;
        endcase
    end
end

endmodule
```

---

## **2.7 RTL Coding Style and Synthesizability**

The design follows strict RTL best practices:

* Fully synchronous design
* Single clock domain
* No behavioral delays (`#`)
* No latch inference
* Clear separation of:

  * Datapath
  * Control path
  * Output logic

The RTL is suitable for:

* Vivado simulation
* RTL schematic generation
* FPGA implementation
* ASIC synthesis (theoretical)

---

## **2.8 Design Characteristics**

* Deterministic timing
* Human-scale control from high-frequency clock
* Easily extendable back to four-way if required
* Fully synchronous RTL
* Moore FSM (stable outputs)
* Counter-based timing
* No behavioral delays
* FPGA-synthesizable
* Simulation-scalable
* Datapath reused from 4-way controller

---

# **PART 3: TESTBENCH (VERIFICATION)**

---

## **3.1 Purpose of the Testbench**

The purpose of the testbench is to **functionally verify** the three-way traffic light controller under different operating conditions.
It ensures that:

* The FSM transitions occur in the correct order
* Timing behavior matches the intended traffic durations
* Blink mode correctly overrides normal operation
* Reset initializes the controller to a safe state

---

## **3.2 Verification Objectives**

The testbench is designed to verify the following:

* Correct sequencing of **main road** and **side road** signals
* Proper activation of **green → yellow → red** phases
* Accurate timing using counter-based delays
* Correct **priority handling of blink mode**
* Reliable recovery from reset

---

## **3.3 Verification Strategy**

A **directed, time-driven verification strategy** is used:

* Apply asynchronous active-low reset
* Allow FSM to run freely in normal traffic mode
* Enable blink mode after sufficient normal operation
* Disable blink mode and observe resumption of normal sequencing
* Observe waveforms for correctness and stability

This approach closely mirrors **real-world operation** of a traffic controller.

---

## **3.4 Simulation Time Scaling Considerations**

### **Need for Time Scaling**

In real hardware:

* Clock frequency = **50 MHz**
* Base time = **0.1 s**, derived using large counters

Direct simulation of real-time seconds is **impractical** due to excessive simulation time and resource usage.

---

### **Simulation Time Scaling Approach**

For simulation:

* Clock period is reduced to **20 ps**
* Base time of **0.1 s** is mapped to **0.1 ns**
* Counters are proportionally scaled

This preserves **functional correctness** while making simulation feasible.

---

### **Real Time vs Simulation Time Mapping**

| Phase           | Real Time | Simulation Time |
| --------------- | --------- | --------------- |
| Base time       | 0.1 s     | 0.1 ns          |
| Main road green | 45 s      | 45 ns           |
| Side road green | 25 s      | 25 ns           |
| Yellow phase    | 5 s       | 5 ns            |
| Blink interval  | 0.5 s     | 0.5 ns          |

> ⚠️ **Important:**
> This scaling affects **only simulation**, not hardware behavior.

---

## **3.5 Testbench Design Considerations**

* Clock is generated continuously (free-running)
* Reset is applied asynchronously
* Blink signal is asserted and deasserted explicitly
* No internal DUT signals are forced
* Verification is based purely on **observed outputs**

---

## **3.6 Testbench code**

```verilog
`timescale 1ns / 1ps
/*---------------------------------------------------------
  Testbench for Three-Way Traffic Light Controller
  File Name : traffic_controller_3way_tb.v

  Simulation Time Scaling:
  - Hardware clock : 50 MHz (20 ns)
  - Simulation clock : 20 ps
  - 0.1 s real time  -> 0.1 ns simulation time
---------------------------------------------------------*/

`define clkperiodby2 0.01   // 10 ps half-period -> 20 ps clock

module traffic_controller_3way_tb;

    /*---------------- Testbench Signals ----------------*/
    reg clk;
    reg reset_n;
    reg blink;

    /* Main road outputs */
    wire MR1, MR2;
    wire MY1, MY2;
    wire MG1, MG2;

    /* Side road outputs */
    wire SR;
    wire SY;
    wire SG;

    /*--------------------------------------------------
      DUT Instantiation
    --------------------------------------------------*/
    traffic_controller_3way dut (
        .clk    (clk),
        .reset_n(reset_n),
        .blink  (blink),

        .MR1 (MR1),
        .MR2 (MR2),
        .MY1 (MY1),
        .MY2 (MY2),
        .MG1 (MG1),
        .MG2 (MG2),

        .SR  (SR),
        .SY  (SY),
        .SG  (SG)
    );

    /*--------------------------------------------------
      Clock Generation (Free-running)
    --------------------------------------------------*/
    always
        #`clkperiodby2 clk = ~clk;

    /*--------------------------------------------------
      Test Sequence
    --------------------------------------------------*/
    initial begin
        /* Initialize signals */
        clk     = 1'b1;
        reset_n = 1'b1;
        blink   = 1'b0;

        /* Apply asynchronous reset */
        #20;
        reset_n = 1'b0;
        #20;
        reset_n = 1'b1;

        /*------------------------------------------------
          Normal Traffic Operation
        ------------------------------------------------*/
        #160;   // Allow multiple traffic phases to execute

        /*------------------------------------------------
          Enable Blink Mode
        ------------------------------------------------*/
        blink = 1'b1;
        #20;    // Observe yellow flashing behavior

        /*------------------------------------------------
          Exit Blink Mode
        ------------------------------------------------*/
        blink = 1'b0;
        #80;    // Resume normal traffic operation

        /* End simulation */
        $stop;
    end

endmodule
```

---

## **3.7 Applied Test Scenarios**

### **Scenario 1: Reset Initialization**

**Inputs Applied**

* `reset_n = 0`
* `blink = 0`

**Expected Behavior**

* All traffic lights OFF
* FSM enters initial safe state
* All counters reset

---

### **Scenario 2: Normal Traffic Operation**

**Inputs Applied**

* `reset_n = 1`
* `blink = 0`

**Expected Behavior**

* Main road green active initially
* Yellow transition after main green
* Side road green phase follows
* Side road yellow before returning to main road
* No conflicting green signals

---

### **Scenario 3: Blink Mode Operation**

**Inputs Applied**

* `blink = 1`

**Expected Behavior**

* Normal FSM operation interrupted
* All green and red lights OFF
* Yellow lights flash periodically
* FSM remains in blink state

---

### **Scenario 4: Exit from Blink Mode**

**Inputs Applied**

* `blink = 0`

**Expected Behavior**

* FSM exits blink mode
* Traffic sequence restarts from initial state
* Normal timing resumes

---

## **3.8 Observability During Simulation**

Key signals observed in the waveform:

### **Clock and Control**

* `clk`
* `reset_n`
* `blink`

### **Main Road Outputs**

* `MG1`, `MG2`
* `MY1`, `MY2`
* `MR1`, `MR2`

### **Side Road Outputs**

* `SG`
* `SY`
* `SR`

### **(Optional – Internal)**

* FSM state
* Timer completion flags

---

## **3.9 Verification Summary**

* FSM sequencing verified under normal and blink modes
* Timing behavior validated using scaled simulation time
* Reset and recovery behavior confirmed
* Design exhibits deterministic, glitch-free operation

---
<img width="1629" height="882" alt="image" src="https://github.com/user-attachments/assets/00c71fc6-b1f1-463e-95f2-2bb93ebf2593" />

<img width="1623" height="878" alt="image" src="https://github.com/user-attachments/assets/4c62c7ea-e76e-46c6-8f9a-83a142c87f04" />
<img width="1626" height="878" alt="image" src="https://github.com/user-attachments/assets/c6371fbc-9440-4083-a677-bc9fc16b13cf" />

---

# **PART 4: SIMULATION RESULTS AND WAVEFORM ANALYSIS**

---
<img width="1629" height="882" alt="image" src="https://github.com/user-attachments/assets/847ad1cd-0b2b-4b91-964e-bb9e2dff8aab" />

## **4.1 Overview of Observed Simulation**

The simulation waveform confirms the correct operation of the **three-way traffic light controller** under the following conditions:

* Normal traffic operation (`blink = 0`)
* Blink mode operation (`blink = 1`)
* Proper reset behavior
* Correct FSM sequencing across all defined states

The observed FSM transitions, timer expirations, and traffic signal activations **align precisely with the designed state machine and timing logic**.

---

## **4.2 Clock and Reset Behavior**

### **Clock (`clk`)**

* Free-running periodic signal
* Drives all counters and FSM state transitions
* Stable and uniform throughout the simulation

### **Reset (`reset_n`)**

* Active-low reset

**When asserted (`reset_n = 0`):**

* FSM state resets to **S0**
* All traffic signals are driven to safe OFF states
* All timer counters are cleared

**When deasserted (`reset_n = 1`):**

* FSM begins normal traffic sequencing
* Timers start operating under FSM control

This behavior is clearly visible at the beginning of the waveform.

---

## **4.3 FSM State Evolution**

The signal `state[3:0]` shows clean and deterministic transitions through the defined states:

| FSM State | Functional Meaning       |
| --------- | ------------------------ |
| S0        | Main road straight green |
| S1        | Main road yellow         |
| S2        | Side road green          |
| S3        | Side road yellow         |
| S12       | Blink mode               |

State transitions occur **only after the corresponding timer completion flag (`res_cnt*`) is asserted**, confirming proper **FSM–datapath synchronization**.

---

## **4.4 Main Road Signal Behavior**

### **Main Green (`MG1`, `MG2`)**

* Asserted during **S0**
* Deasserted before yellow transitions
* Never overlap with side road green signals

### **Main Yellow (`MY1`, `MY2`)**

* Asserted during **S1**
* Active for exactly one yellow-timer duration
* Provide safe transition warning

### **Main Red (`MR1`, `MR2`)**

* Asserted when side road traffic is active
* Prevent conflicting movements

---

## **4.5 Side Road Signal Behavior**

### **Side Green (`SG`)**

* Asserted during **S2**
* Mutually exclusive with main road green

### **Side Yellow (`SY`)**

* Asserted during **S3**
* Visible as short pulses matching the yellow timer

### **Side Red (`SR`)**

* Asserted whenever main road traffic is active
* Guarantees safe isolation of traffic flows

---

## **4.6 Timer Behavior and Control**

The counters (`cnt2_reg`, `cnt3_reg`, `cnt4_reg`, `cnt5_reg`) demonstrate:

* Incrementing **only when their corresponding `start_timer_*` signal is asserted**
* Resetting immediately upon reaching terminal count
* Producing clean `res_cnt*` pulses used by the FSM

This confirms that:

* Timers are **entirely FSM-controlled**
* No unintended free-running timing occurs
* Timing behavior is deterministic and precise

---

## **4.7 Blink Mode Operation**

When `blink = 1`:

* FSM transitions immediately to **S12**
* All green and red signals are turned OFF
* All yellow signals (`MY*`, `SY`) toggle periodically
* Toggle frequency corresponds to the blink timer (0.5 s real time equivalent)

When `blink` returns to `0`:

* FSM exits blink state
* Traffic sequencing restarts from **S0**

This behavior is clearly visible in the later portion of the waveform.

---

## **4.8 Absence of Glitches and Conflicts**

From the waveform analysis:

* No simultaneous green signals in conflicting directions
* No glitches during state transitions
* All outputs change synchronously on clock edges

This validates:

* Proper FSM implementation
* Moore-style output stability
* Correct timer-driven sequencing

---

## **4.9 Summary of Waveform Validation**

✔ Correct reset initialization
✔ Accurate timing using counters
✔ Proper FSM state transitions
✔ Safe traffic signal sequencing
✔ Reliable blink mode operation

**Conclusion:**
The simulation waveform fully validates the **functional correctness, timing accuracy, and robustness** of the three-way traffic light controller design.

---

# Constraints

```verilog
create_clock -period 10 -name clk [get_port clk]

set_property PACKAGE_PIN V11 [get_ports MG1]
set_property PACKAGE_PIN V12 [get_ports MG2]
set_property PACKAGE_PIN V14 [get_ports MR1]
set_property PACKAGE_PIN V15 [get_ports MR2]
set_property PACKAGE_PIN T16 [get_ports MY1]
set_property PACKAGE_PIN U14 [get_ports MY2]
set_property PACKAGE_PIN T15 [get_ports SG]
set_property PACKAGE_PIN V16 [get_ports SR]
set_property PACKAGE_PIN U16 [get_ports SY]
set_property PACKAGE_PIN V10 [get_ports blink]
set_property PACKAGE_PIN E3 [get_ports clk]
set_property PACKAGE_PIN J15 [get_ports reset_n]
```

<img width="1629" height="886" alt="image" src="https://github.com/user-attachments/assets/35dea201-6652-4ea3-8dc8-0e3e044b94fd" />
<img width="1624" height="875" alt="image" src="https://github.com/user-attachments/assets/0087463a-fb16-47b1-9ecd-a9249db00d9b" />



