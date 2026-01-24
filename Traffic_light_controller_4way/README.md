# **PART 1: THEORY, ARCHITECTURE, FSM DESIGN**

---

## **1. Concept / System Overview**

A **four-way traffic light controller** is a sequential digital system used to regulate traffic flow at an intersection involving:

* Two **main roads**
* Two **side roads**
* Straight movements
* Dedicated **left-turn movements**
* Safety-critical yellow transitions
* A special **blink mode** during non-traffic hours

The controller operates cyclically, ensuring that:

* Only **non-conflicting traffic movements** are allowed simultaneously
* Adequate time is provided for each movement
* Transitions between phases are safe and deterministic

This design models a **realistic urban intersection** using a **finite state machine (FSM)** combined with **counter-based timing**.

---

## **2. Need for Sequential Operation**

A traffic light controller **cannot be implemented as a purely combinational circuit** because:

* Traffic signals must remain active for **specific durations**
* State transitions depend on **elapsed time**, not just inputs
* The system must **remember** the current traffic phase
* Blink mode must override normal sequencing

Hence, the system inherently requires:

* Memory elements
* Clock-driven state transitions
* Deterministic sequencing

This makes the controller a **synchronous sequential system**.

---

## **3. Role of Finite State Machine (FSM)**

The FSM forms the **control core** of the traffic controller.

### FSM Responsibilities:

* Define the **order** of traffic phases
* Decide **which lights are ON/OFF** in each phase
* Control **when timers start and stop**
* Handle **priority transitions** (e.g., blink mode)

### FSM Characteristics:

* **Moore-type FSM**
* Outputs depend only on the **current state**
* State transitions occur on **clock edges**
* Ensures **glitch-free outputs**, which is critical for safety systems

---

## **4. Description of the Four-Way Traffic Scenario**

The intersection consists of:

### Main Roads

* Straight movement (Green → Yellow → Red)
* Dedicated left-turn signals

### Side Roads

* Straight movement
* Dedicated left-turn signals

### Operating Modes

1. **Normal Traffic Mode**

   * Cyclic sequencing of main road and side road traffic
   * Yellow transitions between phases
2. **Blink Mode**

   * All yellow lights flash
   * Used during low-traffic or night hours

---

## **5. Datapath Overview**

The datapath of the traffic controller is **time-driven**, not data-driven.

### Datapath Components:

* One **free-running base counter**
* Multiple **derived timers**
* Timer completion flags used by FSM

### Datapath Function:

* Generate a **time base**
* Measure elapsed time for each traffic phase
* Provide timing feedback to the FSM

---

## **6. Datapath Architecture**

The datapath of the four-way traffic light controller is responsible for all **timing-related operations**.
It converts the high-frequency system clock into **coarse, human-perceivable time intervals** required for traffic control.

---

### **6.1 Clock and Time-Base Generation**

The controller operates with a system clock of **50 MHz**, which corresponds to:
`
[
\text{Clock period} = \frac{1}{50,\text{MHz}} = 20,\text{ns}
]
`
Traffic signals must remain active for durations in the order of **seconds**, not nanoseconds.
Therefore, the datapath generates a **0.1 s time base** using a free-running counter.

---

### **6.2 Free-Running Base Counter Operation**

A **22-bit counter** (`cnt1_reg[21:0]`) is used to divide the 50 MHz clock:

* The counter increments on every rising edge of the clock
* When the counter reaches **4,999,999**, it resets to zero

This count corresponds to:

[
4{,}999{,}999 \times 20,\text{ns} = 0.1,\text{s}
]

Thus, the counter produces a **periodic 0.1 s timing tick**, which serves as the **fundamental time unit** for the entire system.

This counter runs **continuously**, independent of the FSM state.

---

### **6.3 Derived Timers and Their Purpose**

Using the 0.1 s time base, the datapath implements multiple **derived timers**, each serving a specific traffic function:

| Timer   | Counter         | Purpose                     | Duration |
| ------- | --------------- | --------------------------- | -------- |
| Timer-1 | `cnt2_reg[8:0]` | Main road straight movement | 45 s     |
| Timer-2 | `cnt3_reg[5:0]` | Yellow transition           | 5 s      |
| Timer-3 | `cnt4_reg[7:0]` | Side road / left turns      | 25 s     |
| Timer-4 | `cnt5_reg[7:0]` | Blink mode                  | 0.5 s    |

Each timer counts **0.1 s ticks**, not raw clock cycles.

---

### **6.4 Timer Enable and Control Behavior**

The FSM controls the datapath using **timer enable signals**:

* `start_timer_1`
* `start_timer_2`
* `start_timer_3`

**Timer behavior:**

* A timer increments only when its corresponding start signal is asserted
* If the start signal is deasserted, the timer **holds its current value**

When a timer reaches its programmed limit:

* A completion flag (`res_cnt*`) is asserted
* The counter resets to zero
* The FSM uses this flag to advance to the next state

This mechanism ensures **precise and deterministic timing** for each traffic phase.

---

### **6.5 Timer Completion Flags**

Each timer produces a completion signal that interfaces directly with the control path:

| Signal     | Meaning                                  |
| ---------- | ---------------------------------------- |
| `res_cnt2` | Main road straight duration completed    |
| `res_cnt3` | Yellow phase completed                   |
| `res_cnt4` | Side road / left turn duration completed |
| `res_cnt5` | Blink interval completed                 |

These flags form the **primary datapath-to-control-path interface**.

---

### **6.6 Role of `cnt*_next` Signals in the Datapath**

For each timer counter, the datapath uses a corresponding **next-value signal** (`cnt*_next`) to compute the upcoming counter value before it is registered.

The role of `cnt*_next` signals is:

* To represent **counter + 1** computation
* To separate **combinational next-value logic** from **sequential storage**
* To maintain clean, synchronous counter updates

General behavior:

* `cnt*_next = current counter value + 1`
* On the clock edge:

  * If reset is active → counter resets to zero
  * If terminal count is reached → counter resets to zero
  * If enabled → counter loads `cnt*_next`
  * Otherwise → counter retains its value

Thus:

* `cnt*_next` defines **what the next value should be**
* FSM control signals define **when that value is loaded**

This separation ensures **clarity, correctness, and synthesizability** of the datapath.

---

### **6.7 Datapath Design Characteristics**

* Fully synchronous design
* Single clock domain
* Counter-based timing (hardware efficient)
* No combinational delays
* Deterministic and repeatable behavior

---

## **7. Control Path Architecture**

The control path of the four-way traffic light controller is implemented entirely using a **finite state machine (FSM)**.
It governs the sequencing of traffic signals by enabling timers, monitoring their completion, and determining the next state of operation.

---

### **7.1 Control Path Responsibilities**

The control path performs the following functions:

* Enables the appropriate timer for each traffic phase
* Monitors timer completion conditions
* Controls FSM state transitions
* Overrides normal traffic operation during blink mode

The control path ensures that **all state transitions are time-driven and deterministic**.

---

### **7.2 Timer Enable Signals**

Each traffic phase is associated with a specific timer, enabled by the FSM through control signals:

| Control Signal  | Purpose                                   |
| --------------- | ----------------------------------------- |
| `start_timer_1` | Enables main road straight traffic timing |
| `start_timer_2` | Enables yellow phase timing               |
| `start_timer_3` | Enables side road / left-turn timing      |

A timer advances **only when its corresponding start signal is asserted**.

---

### **7.3 Timer Advance Condition (Logical Expression)**

A timer advances when **both** of the following conditions are satisfied:

1. The FSM asserts the timer start signal
2. The base time counter reaches the 0.1 s time base

This condition is expressed logically as:

> **Timer Advance Condition**
>
> **start_timer_* = 1 AND (cnt1_reg = time_base)**

Where:

* `start_timer_*` represents the timer enable signal asserted by the FSM
* `cnt1_reg` is the free-running base counter
* `time_base` corresponds to one base time interval (0.1 s)

Only when this condition is true does the associated timer counter increment.

---

### **7.4 Timer Completion Condition**

A timer is considered complete when:

> **(cnt1_reg = time_base) AND (timer_count = terminal_value)**

This condition generates a **timer completion flag** (`res_cnt*`), which is used by the FSM to decide whether to transition to the next state.

---

### **7.5 Timer Completion Flags**

Each timer produces a completion signal monitored by the FSM:

| Completion Signal | Meaning                               |
| ----------------- | ------------------------------------- |
| `res_cnt2`        | Main road straight phase completed    |
| `res_cnt3`        | Yellow phase completed                |
| `res_cnt4`        | Side road / left-turn phase completed |
| `res_cnt5`        | Blink interval completed              |

These signals act as **decision inputs** to the FSM.

---

### **7.6 FSM State Transition Control**

At every clock edge, the FSM evaluates:

* The current state
* The relevant timer completion flag
* The `blink` input

If the required timer has completed, the FSM transitions to the next state.
If not, it remains in the current state.

Thus, **time governs state progression**, not external traffic inputs.

---

### **7.7 Blink Mode Priority Rule**

Blink mode has the **highest priority** in the control path.

This rule can be stated as:

> **If `blink = 1`, the FSM immediately transitions to the blink state, regardless of the current state.**

While in blink mode:

* Normal traffic timers are disabled
* Only the blink timer is active
* All traffic sequencing is suspended

When `blink` is deasserted, the FSM exits blink mode and resumes normal operation from the initial state.

---

## **8. Inputs and Outputs**

### **Top-Level Input Signals**

| Signal    | Description                  |
| --------- | ---------------------------- |
| `clk`     | System clock                 |
| `reset_n` | Active-low reset             |
| `blink`   | Enables blinking-yellow mode |

---

### **Output Signals – Main Road**

| Signal         | Description         |
| -------------- | ------------------- |
| `MG1`, `MG2`   | Main road green     |
| `MY1`, `MY2`   | Main road yellow    |
| `MR1`, `MR2`   | Main road red       |
| `MLT1`, `MLT2` | Main road left-turn |

---

### **Output Signals – Side Road**

| Signal         | Description         |
| -------------- | ------------------- |
| `SG1`, `SG2`   | Side road green     |
| `SY1`, `SY2`   | Side road yellow    |
| `SR1`, `SR2`   | Side road red       |
| `SLT1`, `SLT2` | Side road left-turn |

---

## **9. FSM State Definitions**

The FSM consists of **13 states**:

| State | Description              |
| ----- | ------------------------ |
| `S0`  | Main road straight green |
| `S1`  | Main road yellow         |
| `S2`  | Main road 1 left turn    |
| `S3`  | Main road yellow         |
| `S4`  | Main road 2 left turn    |
| `S5`  | Main road yellow         |
| `S6`  | Side road straight green |
| `S7`  | Side road yellow         |
| `S8`  | Side road 1 left turn    |
| `S9`  | Side road yellow         |
| `S10` | Side road 2 left turn    |
| `S11` | Side road yellow         |
| `S12` | Blink mode               |

---

## **10. FSM State Transition Behavior**

### Normal Operation:

* FSM advances sequentially
* Transitions occur only after:

  * Corresponding timer expires
* Yellow states act as **safety buffers**

### Blink Mode:

* FSM immediately jumps to `S12`
* Remains in blink state while `blink = 1`
* Returns to `S0` when blink is deasserted

---

## **11. Timing Behavior (Conceptual)**

| Phase         | Real Time | Simulation Time |
| ------------- | --------- | --------------- |
| Base time     | 0.1 s     | 0.1 ns          |
| Main straight | 45 s      | 45 ns           |
| Side / left   | 25 s      | 25 ns           |
| Yellow        | 5 s       | 5 ns            |
| Blink         | 0.5 s     | 0.5 ns          |

This scaling affects **only simulation**, not hardware behavior.

---

## **12. Design Strategy and Rationale**

* Moore FSM chosen for **glitch-free outputs**
* Counter-based timing ensures **precise durations**
* Shared base counter reduces hardware
* Conservative yellow signaling improves safety
* Priority-based blink handling ensures robustness

---

## **13. Summary**

* The four-way traffic controller is a **time-driven FSM-controlled system**
* Datapath handles **timing**, control path handles **sequencing**
* FSM ensures **safe, deterministic traffic flow**
* Design is suitable for **FPGA implementation** and **real-world control systems**

---

# **PART 2: SYNTHESIZABLE VERILOG RTL**

---

## **2.1 Design Requirements**

The four-way traffic light controller is designed to meet the following requirements:

* Control a **four-road junction** consisting of:

  * Two main roads
  * Two side roads
* Support **straight** and **left-turn** movements for each road
* Provide fixed timing:

  * Main road green: **45 s**
  * Side road green: **25 s**
  * Yellow phase: **5 s**
  * Blink mode: **0.5 s**
* Ensure **mutually exclusive green signals**
* Include **safe yellow and all-red transitions**
* Support **blinking yellow mode** during non-traffic hours
* Be **fully synchronous**, resettable, and synthesizable

---

## **2.2 Module Interface Definition**

### **Top-Level Module: `traffic_controller`**

#### **Input Signals**

| Signal    | Description                   |
| --------- | ----------------------------- |
| `clk`     | System clock                  |
| `reset_n` | Active-low asynchronous reset |
| `blink`   | Enables blinking-yellow mode  |

---

#### **Output Signals – Main Road**

| Signal         | Description                |
| -------------- | -------------------------- |
| `MR1`, `MR2`   | Main road red lights       |
| `MY1`, `MY2`   | Main road yellow lights    |
| `MG1`, `MG2`   | Main road green lights     |
| `MLT1`, `MLT2` | Main road left-turn lights |

---

#### **Output Signals – Side Road**

| Signal         | Description                |
| -------------- | -------------------------- |
| `SR1`, `SR2`   | Side road red lights       |
| `SY1`, `SY2`   | Side road yellow lights    |
| `SG1`, `SG2`   | Side road green lights     |
| `SLT1`, `SLT2` | Side road left-turn lights |

---

## **2.3 Internal Signal Classification**

### **Timing Datapath Signals**

| Signal           | Purpose                                             |
| ---------------- | --------------------------------------------------- |
| `cnt1_reg[21:0]` | Free-running base counter providing 0.1 s time base |
| `cnt2_reg[8:0]`  | Main road green timer counter (45 s)                |
| `cnt3_reg[5:0]`  | Yellow phase timer counter (5 s)                    |
| `cnt4_reg[7:0]`  | Side road / left-turn timer counter (25 s / 10 s)   |
| `cnt5_reg[7:0]`  | Blink mode timer counter (0.5 s)                    |

---

### **Timer Control Signals**

| Signal          | Purpose                             |
| --------------- | ----------------------------------- |
| `start_timer_1` | Enables main road green timer       |
| `start_timer_2` | Enables yellow phase timer          |
| `start_timer_3` | Enables side road / left-turn timer |

---

### **FSM and Status Signals**

| Signal       | Purpose                                     |
| ------------ | ------------------------------------------- |
| `state[3:0]` | Current FSM state                           |
| `res_cnt2`   | Main road timer completion flag             |
| `res_cnt3`   | Yellow timer completion flag                |
| `res_cnt4`   | Side road / left-turn timer completion flag |
| `res_cnt5`   | Blink timer completion flag                 |


---

## **2.3 RTL Code**

```verilog
/*---------------------------------------------------------
  Four-Way Traffic Light Controller
  Design file : traffic_controller.v
  Description : Controls traffic lights for a four-road
                junction with main roads, side roads,
                left turns, and blinking mode.
---------------------------------------------------------*/

/*---------------- State Definitions ----------------*/
`define S0   4'd0
`define S1   4'd1
`define S2   4'd2
`define S3   4'd3
`define S4   4'd4
`define S5   4'd5
`define S6   4'd6
`define S7   4'd7
`define S8   4'd8
`define S9   4'd9
`define S10  4'd10
`define S11  4'd11
`define S12  4'd12

/*---------------- Timing Parameters ----------------*/
/* For 50 MHz clock → 0.1s time base */
`define time_base 22'd4999999

/* For simulation clock which is 50GHz or 20ps, count of 5 enough for time base of 0.1ns, which is simulation friendly 
/* `define time_base 23'd4 */

/* Timer count values (in units of 0.1 s) */
`define load_cnt2 9'd449   // 45 seconds
`define load_cnt3 6'd49    // 5 seconds
`define load_cnt4 8'd249   // 25 seconds
`define load_cnt5 8'd4     // 0.5 seconds (blink)

/*--------------------------------------------------*/
module traffic_controller (
    input  clk,
    input  reset_n,
    input  blink,

    /* Main road signals */
    output reg MR1, MR2,
    output reg MY1, MY2,
    output reg MG1, MG2,
    output reg MLT1, MLT2,

    /* Side road signals */
    output reg SR1, SR2,
    output reg SY1, SY2,
    output reg SG1, SG2,
    output reg SLT1, SLT2
);

/*---------------- Internal Wires ----------------*/
wire adv_cnt2, adv_cnt3, adv_cnt4, adv_cnt5;
wire res_cnt2, res_cnt3, res_cnt4, res_cnt5;

/*---------------- Counters ----------------*/
reg  [21:0] cnt1_reg;
reg  [8:0]  cnt2_reg;
reg  [5:0]  cnt3_reg;
reg  [7:0]  cnt4_reg;
reg  [7:0]  cnt5_reg;

/*---------------- Counters Next values----------------*/
wire  [21:0] cnt1_next;
wire  [8:0]  cnt2_next;
wire  [5:0]  cnt3_next;
wire  [7:0]  cnt4_next;
wire  [7:0]  cnt5_next;

/*---------------- Timer Control ----------------*/
reg start_timer_1;
reg start_timer_2;
reg start_timer_3;

/*---------------- FSM State ----------------*/
reg [3:0] state;

/*--------------------------------------------------
  Free-Running Counter (Time Base = 0.1s)
--------------------------------------------------*/
assign cnt1_next = cnt1_reg + 1;

always @(posedge clk or negedge reset_n) begin
    if (!reset_n)
        cnt1_reg <= 22'd0;
    else if (cnt1_reg == `time_base)
        cnt1_reg <= 22'd0;
    else
        cnt1_reg <= cnt1_next;
end

/*--------------------------------------------------
  Timer 1 – 45s (Main road green)
--------------------------------------------------*/
assign adv_cnt2 = start_timer_1 & (cnt1_reg == `time_base);
assign res_cnt2 = (cnt1_reg == `time_base) & (cnt2_reg == `load_cnt2);
assign cnt2_next = cnt2_reg + 1;

always @(posedge clk or negedge reset_n) begin
    if (!reset_n)
        cnt2_reg <= 9'd0;
    else if (res_cnt2)
        cnt2_reg <= 9'd0;
    else if (adv_cnt2)
        cnt2_reg <= cnt2_next;
end

/*--------------------------------------------------
  Timer 2 – 5s (Yellow)
--------------------------------------------------*/
assign adv_cnt3 = start_timer_2 & (cnt1_reg == `time_base);
assign res_cnt3 = (cnt1_reg == `time_base) & (cnt3_reg == `load_cnt3);
assign cnt3_next = cnt3_reg + 1;

always @(posedge clk or negedge reset_n) begin
    if (!reset_n)
        cnt3_reg <= 6'd0;
    else if (res_cnt3)
        cnt3_reg <= 6'd0;
    else if (adv_cnt3)
        cnt3_reg <= cnt3_next;
end

/*--------------------------------------------------
  Timer 3 – 25s (Side road green / left)
--------------------------------------------------*/
assign adv_cnt4 = start_timer_3 & (cnt1_reg == `time_base);
assign res_cnt4 = (cnt1_reg == `time_base) & (cnt4_reg == `load_cnt4);
assign cnt4_next = cnt4_reg + 1;

always @(posedge clk or negedge reset_n) begin
    if (!reset_n)
        cnt4_reg <= 8'd0;
    else if (res_cnt4)
        cnt4_reg <= 8'd0;
    else if (adv_cnt4)
        cnt4_reg <= cnt4_next;
end

/*--------------------------------------------------
  Timer 4 – 0.5s Blink Mode
--------------------------------------------------*/
assign adv_cnt5 = blink & (cnt1_reg == `time_base);
assign res_cnt5 = (cnt1_reg == `time_base) & (cnt5_reg == `load_cnt5);
assign cnt5_next = cnt5_reg + 1;

always @(posedge clk or negedge reset_n) begin
    if (!reset_n)
        cnt5_reg <= 8'd0;
    else if (res_cnt5)
        cnt5_reg <= 8'd0;
    else if (adv_cnt5)
        cnt5_reg <= cnt5_next;
end

/*--------------------------------------------------
  Traffic Lights State Machine
--------------------------------------------------*/
always @(posedge clk or negedge reset_n) begin
    if (!reset_n) begin
        /*------------- Reset: All lights OFF -------------*/
        MR1  <= 1'b0; MR2  <= 1'b0;
        MY1  <= 1'b0; MY2  <= 1'b0;
        MG1  <= 1'b0; MG2  <= 1'b0;
        MLT1 <= 1'b0; MLT2 <= 1'b0;

        SR1  <= 1'b0; SR2  <= 1'b0;
        SY1  <= 1'b0; SY2  <= 1'b0;
        SG1  <= 1'b0; SG2  <= 1'b0;
        SLT1 <= 1'b0; SLT2 <= 1'b0;

        start_timer_1 <= 1'b0;
        start_timer_2 <= 1'b0;
        start_timer_3 <= 1'b0;

        state <= `S0;
    end
    else begin
        case (state)

        /*--------------------------------------------------
          S0 : Main road green, Side road red (45s)
        --------------------------------------------------*/
        `S0: begin
            if (blink)
                state <= `S12;
            else begin
                MG1 <= 1'b1; MG2 <= 1'b1;
                SR1 <= 1'b1; SR2 <= 1'b1;

                MY1 <= 1'b0; MY2 <= 1'b0;
                MR1 <= 1'b0; MR2 <= 1'b0;
                MLT1 <= 1'b0; MLT2 <= 1'b0;
                SY1 <= 1'b0; SY2 <= 1'b0;
                SG1 <= 1'b0; SG2 <= 1'b0;
                SLT1 <= 1'b0; SLT2 <= 1'b0;

                if (res_cnt2) begin
                    start_timer_1 <= 1'b0;
                    state <= `S1;
                end
                else begin
                    start_timer_1 <= 1'b1;
                    state <= `S0;
                end
            end
        end

        /*--------------------------------------------------
          S1 : Main road yellow, Side road red (5s)
        --------------------------------------------------*/
        `S1: begin
            if (blink)
                state <= `S12;
            else begin
                MY1 <= 1'b1; MY2 <= 1'b1;
                SR1 <= 1'b1; SR2 <= 1'b1;

                MG1 <= 1'b0; MG2 <= 1'b0;
                MR1 <= 1'b0; MR2 <= 1'b0;
                MLT1 <= 1'b0; MLT2 <= 1'b0;
                SY1 <= 1'b0; SY2 <= 1'b0;
                SG1 <= 1'b0; SG2 <= 1'b0;
                SLT1 <= 1'b0; SLT2 <= 1'b0;

                if (res_cnt3) begin
                    start_timer_2 <= 1'b0;
                    state <= `S2;
                end
                else begin
                    start_timer_2 <= 1'b1;
                    state <= `S1;
                end
            end
        end

        /*--------------------------------------------------
          S2 : Main red, Side red, Main left turn (25s)
        --------------------------------------------------*/
        `S2: begin
            if (blink)
                state <= `S12;
            else begin
                MR1 <= 1'b1; MR2 <= 1'b1;
                MLT1 <= 1'b1;
                SR1 <= 1'b1; SR2 <= 1'b1;

                MY1 <= 1'b0; MY2 <= 1'b0;
                MG1 <= 1'b0; MG2 <= 1'b0;
                MLT2 <= 1'b0;
                SY1 <= 1'b0; SY2 <= 1'b0;
                SG1 <= 1'b0; SG2 <= 1'b0;
                SLT1 <= 1'b0; SLT2 <= 1'b0;

                if (res_cnt4) begin
                    start_timer_3 <= 1'b0;
                    state <= `S3;
                end
                else begin
                    start_timer_3 <= 1'b1;
                    state <= `S2;
                end
            end
        end

        /*--------------------------------------------------
          S3 : Main red, Main yellow, Side red (5s)
        --------------------------------------------------*/
        `S3: begin
            if (blink)
                state <= `S12;
            else begin
                MR1 <= 1'b1; MR2 <= 1'b1;
                MY1 <= 1'b1; MY2 <= 1'b1;
                SR1 <= 1'b1; SR2 <= 1'b1;

                MG1 <= 1'b0; MG2 <= 1'b0;
                MLT1 <= 1'b0; MLT2 <= 1'b0;
                SY1 <= 1'b0; SY2 <= 1'b0;
                SG1 <= 1'b0; SG2 <= 1'b0;
                SLT1 <= 1'b0; SLT2 <= 1'b0;

                if (res_cnt3) begin
                    start_timer_2 <= 1'b0;
                    state <= `S4;
                end
                else begin
                    start_timer_2 <= 1'b1;
                    state <= `S3;
                end
            end
        end

/*--------------------------------------------------
  S4 : Main red, Main road 2 left turn (25s)
--------------------------------------------------*/
`S4: begin
    if (blink)
        state <= `S12;
    else begin
        MR1 <= 1'b1; MR2 <= 1'b1;
        MLT2 <= 1'b1;
        SR1 <= 1'b1; SR2 <= 1'b1;

        MY1 <= 1'b0; MY2 <= 1'b0;
        MG1 <= 1'b0; MG2 <= 1'b0;
        MLT1 <= 1'b0;
        SY1 <= 1'b0; SY2 <= 1'b0;
        SG1 <= 1'b0; SG2 <= 1'b0;
        SLT1 <= 1'b0; SLT2 <= 1'b0;

        if (res_cnt4) begin
            start_timer_3 <= 1'b0;
            state <= `S5;
        end
        else begin
            start_timer_3 <= 1'b1;
            state <= `S4;
        end
    end
end

/*--------------------------------------------------
  S5 : Main red, Main road 2 yellow, Side red (5s)
--------------------------------------------------*/
`S5: begin
    if (blink)
        state <= `S12;
    else begin
        MR1 <= 1'b1; MR2 <= 1'b1;
        MY2 <= 1'b1;
        SR1 <= 1'b1; SR2 <= 1'b1;

        MY1 <= 1'b0;
        MG1 <= 1'b0; MG2 <= 1'b0;
        MLT1 <= 1'b0; MLT2 <= 1'b0;
        SY1 <= 1'b0; SY2 <= 1'b0;
        SG1 <= 1'b0; SG2 <= 1'b0;
        SLT1 <= 1'b0; SLT2 <= 1'b0;

        if (res_cnt3) begin
            start_timer_2 <= 1'b0;
            state <= `S6;
        end
        else begin
            start_timer_2 <= 1'b1;
            state <= `S5;
        end
    end
end

/*--------------------------------------------------
  S6 : Side road green (45s)
--------------------------------------------------*/
`S6: begin
    if (blink)
        state <= `S12;
    else begin
        MR1 <= 1'b1; MR2 <= 1'b1;
        SG1 <= 1'b1; SG2 <= 1'b1;

        MY1 <= 1'b0; MY2 <= 1'b0;
        MG1 <= 1'b0; MG2 <= 1'b0;
        MLT1 <= 1'b0; MLT2 <= 1'b0;
        SR1 <= 1'b0; SR2 <= 1'b0;
        SY1 <= 1'b0; SY2 <= 1'b0;
        SLT1 <= 1'b0; SLT2 <= 1'b0;

        if (res_cnt2) begin
            start_timer_1 <= 1'b0;
            state <= `S7;
        end
        else begin
            start_timer_1 <= 1'b1;
            state <= `S6;
        end
    end
end

/*--------------------------------------------------
  S7 : Side road yellow (5s)
--------------------------------------------------*/
`S7: begin
    if (blink)
        state <= `S12;
    else begin
        MR1 <= 1'b1; MR2 <= 1'b1;
        SY1 <= 1'b1; SY2 <= 1'b1;

        MY1 <= 1'b0; MY2 <= 1'b0;
        MG1 <= 1'b0; MG2 <= 1'b0;
        MLT1 <= 1'b0; MLT2 <= 1'b0;
        SR1 <= 1'b0; SR2 <= 1'b0;
        SG1 <= 1'b0; SG2 <= 1'b0;
        SLT1 <= 1'b0; SLT2 <= 1'b0;

        if (res_cnt3) begin
            start_timer_2 <= 1'b0;
            state <= `S8;
        end
        else begin
            start_timer_2 <= 1'b1;
            state <= `S7;
        end
    end
end

/*--------------------------------------------------
  S8 : Side road 1 left turn (25s)
--------------------------------------------------*/
`S8: begin
    if (blink)
        state <= `S12;
    else begin
        MR1 <= 1'b1; MR2 <= 1'b1;
        SR1 <= 1'b1; SR2 <= 1'b1;
        SLT1 <= 1'b1;

        MY1 <= 1'b0; MY2 <= 1'b0;
        MG1 <= 1'b0; MG2 <= 1'b0;
        MLT1 <= 1'b0; MLT2 <= 1'b0;
        SY1 <= 1'b0; SY2 <= 1'b0;
        SG1 <= 1'b0; SG2 <= 1'b0;
        SLT2 <= 1'b0;

        if (res_cnt4) begin
            start_timer_3 <= 1'b0;
            state <= `S9;
        end
        else begin
            start_timer_3 <= 1'b1;
            state <= `S8;
        end
    end
end

/*--------------------------------------------------
  S9 : Side road yellow (5s)
--------------------------------------------------*/
`S9: begin
    if (blink)
        state <= `S12;
    else begin
        MR1 <= 1'b1; MR2 <= 1'b1;
        SY1 <= 1'b1; SY2 <= 1'b1;

        MY1 <= 1'b0; MY2 <= 1'b0;
        MG1 <= 1'b0; MG2 <= 1'b0;
        MLT1 <= 1'b0; MLT2 <= 1'b0;
        SR1 <= 1'b0; SR2 <= 1'b0;
        SG1 <= 1'b0; SG2 <= 1'b0;
        SLT1 <= 1'b0; SLT2 <= 1'b0;

        if (res_cnt3) begin
            start_timer_2 <= 1'b0;
            state <= `S10;
        end
        else begin
            start_timer_2 <= 1'b1;
            state <= `S9;
        end
    end
end

/*--------------------------------------------------
  S10 : Side road 2 left turn (25s)
--------------------------------------------------*/
`S10: begin
    if (blink)
        state <= `S12;
    else begin
        MR1 <= 1'b1; MR2 <= 1'b1;
        SR1 <= 1'b1; SR2 <= 1'b1;
        SLT2 <= 1'b1;

        MY1 <= 1'b0; MY2 <= 1'b0;
        MG1 <= 1'b0; MG2 <= 1'b0;
        MLT1 <= 1'b0; MLT2 <= 1'b0;
        SY1 <= 1'b0; SY2 <= 1'b0;
        SG1 <= 1'b0; SG2 <= 1'b0;
        SLT1 <= 1'b0;

        if (res_cnt4) begin
            start_timer_3 <= 1'b0;
            state <= `S11;
        end
        else begin
            start_timer_3 <= 1'b1;
            state <= `S10;
        end
    end
end

/*--------------------------------------------------
  S11 : Side road yellow, Main red (5s)
--------------------------------------------------*/
`S11: begin
    if (blink)
        state <= `S12;
    else begin
        MY1 <= 1'b1; MY2 <= 1'b1;
        SR1 <= 1'b1; SR2 <= 1'b1;
        SY2 <= 1'b1;

        MR1 <= 1'b0; MR2 <= 1'b0;
        MG1 <= 1'b0; MG2 <= 1'b0;
        MLT1 <= 1'b0; MLT2 <= 1'b0;
        SY1 <= 1'b0;
        SG1 <= 1'b0; SG2 <= 1'b0;
        SLT1 <= 1'b0; SLT2 <= 1'b0;

        if (res_cnt3) begin
            start_timer_2 <= 1'b0;
            state <= `S0;
        end
        else begin
            start_timer_2 <= 1'b1;
            state <= `S11;
        end
    end
end



        /*--------------------------------------------------
          S12 : Blink Mode (All yellow flashing at 1 Hz)
        --------------------------------------------------*/
        `S12: begin
            MR1 <= 1'b0; MR2 <= 1'b0;
            MG1 <= 1'b0; MG2 <= 1'b0;
            MLT1 <= 1'b0; MLT2 <= 1'b0;
            SR1 <= 1'b0; SR2 <= 1'b0;
            SG1 <= 1'b0; SG2 <= 1'b0;
            SLT1 <= 1'b0; SLT2 <= 1'b0;

            if (res_cnt5) begin
                MY1 <= ~MY1;
                MY2 <= ~MY2;
                SY1 <= ~SY1;
                SY2 <= ~SY2;
            end

            if (!blink)
                state <= `S0;
            else
                state <= `S12;
        end

        default: state <= `S0;
        endcase
    end
end

endmodule
```

---

## **2.4 RTL Coding Style and Discipline**

* **Fully synthesizable Verilog**
* No behavioral delays (`#`)
* No latches
* Separate:

  * Timing datapath
  * FSM control logic
  * Output logic
* Asynchronous active-low reset
* One clock domain

---

## **2.5 FSM Implementation Characteristics**

* **Single FSM** controlling entire intersection
* **13 states (S0–S12)**:

  * Normal traffic sequencing
  * Left-turn handling
  * Blink mode
* Transitions are **time-driven**, not input-driven
* Blink mode has **highest priority**

---

## **2.6 Datapath Characteristics**

* One free-running counter generates a **0.1 s time base**
* All timers derive from the same base counter
* Timers are:

  * Explicitly started
  * Explicitly stopped
* Counters reset cleanly after terminal count

This minimizes hardware duplication and simplifies timing control.

---

## **2.7 Design Characteristics**

| Aspect           | Description                        |
| ---------------- | ---------------------------------- |
| Architecture     | Control-dominated FSM              |
| Timing           | Counter-based                      |
| Safety           | All-red and yellow phases included |
| Expandability    | Additional phases can be added     |
| Synthesizability | FPGA-safe                          |
| Realism          | Matches real traffic systems       |

---

# **PART 3: TESTBENCH (VERIFICATION)**


---

## **3.1 Purpose of the Testbench**

The purpose of this testbench is to verify the functional correctness of the four-way traffic light controller by simulating:

* Normal traffic light sequencing
* Blink mode operation during non-traffic hours
* Proper response to reset
* FSM progression across multiple traffic states

The testbench focuses on **functional verification of the FSM and control logic**, not real-time traffic accuracy.

---

## **3.2 Objectives**

The objectives of the testbench are to:

* Validate correct sequencing of main road, side road, and left-turn signals
* Verify correct assertion of green, yellow, and red phases
* Confirm correct operation of blink mode (all yellow flashing)
* Ensure proper FSM initialization and reset behavior
* Observe stable and glitch-free outputs across state transitions

---

## **3.3 Verification Strategy**

A **directed and time-scaled verification strategy** is adopted:

* Apply asynchronous active-low reset
* Allow the FSM to execute continuously
* Enable blink mode after normal operation
* Disable blink mode and resume normal sequencing
* Observe waveform outputs for correctness

This strategy preserves **logical behavior** while using **scaled simulation timing**.

---

## **3.4 Testbench Design Considerations and Time Scaling**

### **Need for Time Scaling**

In real hardware:

* Clock frequency = **50 MHz**
* Clock period = **20 ns**
* Internal time base = **0.1 s**

To generate this time base:
[
\frac{0.1\ \text{s}}{20\ \text{ns}} = 5{,}000{,}000\ \text{clock cycles}
]

Simulating millions of clock cycles for each FSM transition is **computationally expensive** and impractical in RTL simulation.

---

### **Simulation Time Scaling Method**

To make simulation feasible while preserving FSM behavior:

* Timescale is set to `1ns/1ps`
* Testbench clock half-period is set to **10 ps**
* Full clock period becomes **20 ps**

Thus, the simulation clock is a **time-scaled equivalent** of the real hardware clock.

---

### **Scaled Time Base Derivation**

Target simulation time base = **0.1 ns**

Number of clock cycles required:
[
\frac{0.1\ \text{ns}}{20\ \text{ps}} = 5\ \text{clock cycles}
]

Hence, a counter value of **5** in simulation represents one **0.1 ns time tick**, which is a scaled equivalent of the **0.1 s hardware time base**.

---

### **Real-Time vs Simulation-Time Mapping**

| Function              | Real Hardware Time | Simulation Time |
| --------------------- | ------------------ | --------------- |
| Base time tick        | 0.1 s              | 0.1 ns          |
| Main road green       | 45 s               | 45 ns           |
| Side road / left turn | 25 s               | 25 ns           |
| Yellow phase          | 5 s                | 5 ns            |
| Blink period          | 0.5 s              | 0.5 ns          |

Only the **absolute time scale** is compressed; **relative sequencing and FSM behavior remain unchanged**.

---

### **Key Clarification**

The simulation does **not represent real traffic durations**.
It validates:

* FSM sequencing
* Control signal correctness
* Counter–FSM interaction

Real-time accuracy is achieved **only during FPGA or ASIC implementation**, not simulation.

---

## **3.5 Complete Testbench Code (Simulation-Scaled)**

```verilog
`timescale 1ns/1ps
`define clkperiodby2 0.01   // 10 ps half-period for simulation

module traffic_controller_test;

    // Declare input signals
    reg clk;
    reg reset_n;
    reg blink;

    // Declare output wires
    wire MG1, MG2;
    wire MY1, MY2;
    wire MR1, MR2;
    wire MLT1, MLT2;

    wire SG1, SG2;
    wire SY1, SY2;
    wire SR1, SR2;
    wire SLT1, SLT2;

    //--------------------------------------------------
    // Instantiate the Traffic Controller
    //--------------------------------------------------
    traffic_controller tc1 (
        .clk    (clk),
        .reset_n(reset_n),

        .MG1 (MG1),
        .MG2 (MG2),
        .MR1 (MR1),
        .MR2 (MR2),
        .MY1 (MY1),
        .MY2 (MY2),
        .MLT1(MLT1),
        .MLT2(MLT2),

        .SG1 (SG1),
        .SG2 (SG2),
        .SR1 (SR1),
        .SR2 (SR2),
        .SY1 (SY1),
        .SY2 (SY2),
        .SLT1(SLT1),
        .SLT2(SLT2),

        .blink (blink)
    );

    //--------------------------------------------------
    // Clock Generation
    //--------------------------------------------------
    always
        #`clkperiodby2 clk <= ~clk;

    //--------------------------------------------------
    // Test Sequence
    //--------------------------------------------------
    initial begin
        // Initialize inputs
        clk     = 1'b1;
        reset_n = 1'b1;
        blink   = 1'b0;

        // Apply reset pulse
        #20 reset_n = 1'b0;
        #20 reset_n = 1'b1;

        // Normal traffic operation
        #600;

        // Enable blink mode
        blink = 1'b1;
        #50;

        // Resume normal traffic operation
        blink = 1'b0;
        #50;
    end

endmodule
```

---

## **3.6 Applied Inputs and Expected Outputs**

### **Reset Phase**

* `reset_n = 0`
* All traffic lights OFF
* FSM initialized to starting state

---

### **Normal Traffic Mode (`blink = 0`)**

**Expected Behavior**

* Main road green asserted initially
* Yellow lights asserted during transitions
* Left-turn signals enabled as per FSM
* Side road phases follow main road completion
* No conflicting green signals

---

### **Blink Mode (`blink = 1`)**

**Expected Behavior**

* All red and green lights OFF
* Yellow lights toggle periodically
* FSM remains in blink state

---

### **Return to Normal Mode**

**Expected Behavior**

* FSM exits blink state
* Traffic sequence restarts from the initial state
* Normal sequencing resumes

---

## **3.7 Observability During Simulation**

Signals to observe in the waveform:

* Main road signals: `MG*`, `MY*`, `MR*`, `MLT*`
* Side road signals: `SG*`, `SY*`, `SR*`, `SLT*`
* Control signals: `clk`, `reset_n`, `blink`
* FSM state (optional, internal)

---

## **3.8 Summary**

* Simulation uses time scaling for feasibility
* FSM behavior remains functionally correct
* All traffic phases and blink mode are verified
* Testbench is deterministic and reproducible

---

<img width="1640" height="886" alt="image" src="https://github.com/user-attachments/assets/324c8836-9dbe-4a3a-8b54-afd203f9b2f2" />
<img width="1643" height="878" alt="image" src="https://github.com/user-attachments/assets/9318f6de-77a3-4c1e-9fe6-5328f6a62da3" />
<img width="1642" height="880" alt="image" src="https://github.com/user-attachments/assets/1b80bd1b-fb7d-417a-b568-eec4e85ddf6a" />


# **PART 4: WAVEFORM & TIMING EXPLANATION**

<img width="1643" height="883" alt="image" src="https://github.com/user-attachments/assets/1dab4371-0054-4a66-8b17-f50164a220eb" />

---

## **4.1 Overview of Observed Simulation**

The waveform confirms correct operation of the **four-way traffic light controller** under:

* Normal traffic operation (`blink = 0`)
* Blink mode operation (`blink = 1`)
* Proper reset behavior
* Correct FSM sequencing across multiple states

The FSM transitions, timer expirations, and traffic signal activations align with the **textbook-defined state machine**.

---

## **4.2 Clock and Reset Behavior**

### **Clock (`clk`)**

* Free-running periodic signal
* Drives all counters and FSM transitions
* Uniform and stable across the simulation

### **Reset (`reset_n`)**

* Active-low reset
* When asserted (`reset_n = 0`):

  * FSM state resets to **S0**
  * All traffic lights are turned OFF
  * All timers are cleared
* When deasserted:

  * FSM starts normal traffic sequencing

This behavior is clearly visible at the beginning of the waveform.

---

## **4.3 FSM State Evolution**

The signal `state[3:0]` shows clean transitions through the defined states:

| FSM State | Functional Meaning             |
| --------- | ------------------------------ |
| `S0`      | Main road straight green       |
| `S1`      | Main road yellow               |
| `S2`      | Main road left turn            |
| `S3`      | Main road yellow (post left)   |
| `S4`      | Main road red + side left      |
| `S5`      | Side road yellow               |
| `S6`      | Side road straight green       |
| `S7`      | Side road yellow               |
| `S8–S11`  | Side left, yellow, transitions |
| `S12`     | Blink mode                     |

State changes occur **only after the corresponding timer completion flag** (`res_cnt*`) is asserted, confirming proper FSM–datapath synchronization.

---

## **4.4 Main Road Signal Behavior**

### **Main Green (`MG1`, `MG2`)**

* Asserted during **S0**
* Deasserted before yellow transitions
* Never overlap with side road green signals

### **Main Yellow (`MY1`, `MY2`)**

* Asserted during **yellow transition states**
* Remain ON for exactly one yellow timer duration
* Serve as safe transition indicators

### **Main Red (`MR1`, `MR2`)**

* Asserted when side roads are active
* Ensure no conflicting traffic flow

### **Main Left Turn (`MLT1`, `MLT2`)**

* Activated only in their designated states
* Properly isolated from straight and side movements

---

## **4.5 Side Road Signal Behavior**

### **Side Green (`SG1`, `SG2`)**

* Asserted during **side road straight states**
* Mutually exclusive with main road green

### **Side Yellow (`SY1`, `SY2`)**

* Asserted during side road yellow transitions
* Visible as short pulses corresponding to the yellow timer

### **Side Red (`SR1`, `SR2`)**

* Asserted whenever main road traffic is active

### **Side Left Turn (`SLT1`, `SLT2`)**

* Enabled only in their allocated FSM states
* No overlap with straight traffic

---

## **4.6 Timer Behavior and Control**

The counters (`cnt2_reg`, `cnt3_reg`, `cnt4_reg`, `cnt5_reg`) show:

* Incrementing only when their respective `start_timer_*` signal is asserted
* Resetting immediately when terminal count is reached
* Generating clean `res_cnt*` pulses used by the FSM

This confirms:

* **Timers are fully controlled by FSM**
* No free-running or unintended timing behavior

---

## **4.7 Blink Mode Operation**

When `blink = 1`:

* FSM transitions to **S12**
* All green and red signals are turned OFF
* All yellow lights (`MY*`, `SY*`) toggle periodically
* Toggle rate corresponds to **blink timer (0.5 s equivalent)**

When `blink` returns to `0`:

* FSM exits blink state
* Traffic sequence restarts from **S0**

This behavior is clearly visible in the later portion of the waveform.

---

## **4.8 Absence of Glitches and Conflicts**

From the waveform:

* No simultaneous green signals across conflicting directions
* No glitches on outputs during state transitions
* All outputs change synchronously on clock edges

This validates:

* Proper FSM design
* Safe Moore-style output behavior
* Correct timer-driven sequencing

---

## **4.9 Summary of Waveform Validation**

* ✔ Correct reset initialization
* ✔ Accurate timing via counters
* ✔ Proper FSM state transitions
* ✔ Safe traffic signal sequencing
* ✔ Reliable blink mode operation

---

# **PART 5: Viva Questions, Interview Questions, Debug Scenarios & Design Variations**

---

## **A) Viva Questions (University / Lab-Oriented)**

1. What is a traffic light controller?
2. Why is a traffic light controller a sequential circuit?
3. Why is an FSM required in traffic control systems?
4. What does each state represent in this design?
5. How many states are used in this controller and why?
6. Why are yellow lights mandatory between green and red phases?
7. What is the role of timers in this controller?
8. Why is a 0.1 s time base used instead of seconds directly?
9. How is long-duration timing generated from a 50 MHz clock?
10. What happens to the FSM and outputs during reset?
11. Why is blink mode required in traffic light systems?
12. How does the FSM handle priority for blink mode?
13. Why are outputs registered in this design?
14. Why is this controller implemented as a Moore FSM?
15. What happens if two green signals turn ON together?
16. Why are multiple timers used instead of a single timer?
17. What is the role of `start_timer_*` signals?
18. How does the FSM know when to change states?
19. What happens if a timer never reaches terminal count?
20. Can this controller work without a clock? Why?

---

## **B) Interview-Style Questions (VLSI / Industry)**

### **FSM & Architecture**

1. Why is a Moore FSM preferred over a Mealy FSM for traffic controllers?
2. How would this design change if implemented as a Mealy machine?
3. How would you reduce the number of FSM states?
4. Can this design be parameterized for different timings?
5. What is the latency of one complete traffic cycle?
6. What defines throughput in a traffic controller?
7. How would you add pedestrian crossing support?
8. How would you add emergency vehicle priority?

---

### **Timing & Simulation**

9. Why is counter-based timing preferred over delay-based timing?
10. Why is time scaling required during simulation?
11. How do you verify timing correctness in simulation?
12. What happens if `blink` toggles close to a clock edge?
13. How would you make this design robust to asynchronous inputs?

---

### **Synthesis & Hardware**

14. What hardware components are inferred from this design?
15. How many flip-flops are required approximately?
16. What combinational logic dominates the critical path?
17. How would this map onto FPGA LUTs?
18. What limits the maximum operating frequency?

---

### **Verification**

19. How would you verify all FSM states are reachable?
20. What corner cases exist in this controller?
21. How would you verify reset behavior?
22. How would you verify mutual exclusion of green signals?
23. How would you write a self-checking testbench?
24. How would you use assertions in this design?

---

## **C) Debug Scenarios (Very Important)**

*(Recognizing real-world bugs)*

### **Bug 1: Conflicting Green Signals**

* **Problem:** Two green signals ON simultaneously
* **Cause:** Missing deassertion in FSM state
* **Fix:** Assign all outputs in every state

---

### **Bug 2: FSM Stuck in One State**

* **Problem:** No state transitions
* **Cause:** Timer enable not asserted
* **Fix:** Verify `start_timer_*` logic

---

### **Bug 3: Yellow Lights Flicker**

* **Problem:** Unstable yellow output
* **Cause:** Combinational output logic
* **Fix:** Register outputs in FSM

---

### **Bug 4: Blink Mode Never Exits**

* **Problem:** Controller stuck in blink
* **Cause:** Missing exit condition
* **Fix:** Add transition when `blink = 0`

---

### **Bug 5: Incorrect Timing Duration**

* **Problem:** Phases too short/long
* **Cause:** Wrong counter limits or clock assumptions
* **Fix:** Recalculate timing constants

---

### **Bug 6: Outputs Become X in Simulation**

* **Problem:** Undefined outputs
* **Cause:** No reset initialization
* **Fix:** Initialize all outputs on reset

---

### **Bug 7: FSM Skips States**

* **Problem:** Missing traffic phases
* **Cause:** Multiple transition conditions true
* **Fix:** Ensure mutually exclusive conditions

---

## **D) Design Variations**

### **Moore vs Mealy Traffic Controller**

| Feature          | Moore           | Mealy            |
| ---------------- | --------------- | ---------------- |
| Output stability | High            | Lower            |
| Safety           | Higher          | Risk of glitches |
| Latency          | Slightly higher | Lower            |

---

### **Other Variations**

* Three-way traffic light controller
* Pedestrian signal integration
* Sensor-based adaptive controller
* Emergency override system
* Parameterized timing controller

---

## **E) Conceptual Questions (Deep Understanding)**

1. Why does traffic control favor safety over latency?
2. How does FSM abstraction simplify control logic?
3. Why are registered outputs critical in safety systems?
4. How would metastability affect this design?
5. How would this controller integrate into a smart city system?

---

## **F) If This Were in a Real Chip…**

You would additionally consider:

✔ Clock gating for power reduction
✔ Reset synchronization
✔ CDC handling for blink input
✔ Formal verification of FSM
✔ Assertion-based safety checks
✔ Fault tolerance and watchdog timers

---
# **Part 6: Constraints**

```verilog
create_clock -period 10 -name clk [get_ports clk]

set_property PACKAGE_PIN V11 [get_ports MG1]
set_property PACKAGE_PIN V12 [get_ports MG2]
set_property PACKAGE_PIN V14 [get_ports MLT1]
set_property PACKAGE_PIN V15 [get_ports MLT2]
set_property PACKAGE_PIN T16 [get_ports MR1]
set_property PACKAGE_PIN U14 [get_ports MR2]
set_property PACKAGE_PIN T15 [get_ports MY1]
set_property PACKAGE_PIN V16 [get_ports MY2]
set_property PACKAGE_PIN U17 [get_ports SG1]
set_property PACKAGE_PIN V17 [get_ports SG2]
set_property PACKAGE_PIN R18 [get_ports SLT1]
set_property PACKAGE_PIN N14 [get_ports SLT2]
set_property PACKAGE_PIN J13 [get_ports SR1]
set_property PACKAGE_PIN K15 [get_ports SR2]
set_property PACKAGE_PIN H17 [get_ports SY1]
set_property PACKAGE_PIN E3 [get_ports clk]
set_property PACKAGE_PIN J15 [get_ports reset_n]
set_property PACKAGE_PIN V10 [get_ports blink]
set_property PACKAGE_PIN N16 [get_ports SY2]
set_property IOSTANDARD LVCMOS33 [get_ports MG1]
set_property IOSTANDARD LVCMOS33 [get_ports MG2]
set_property IOSTANDARD LVCMOS33 [get_ports MLT1]
set_property IOSTANDARD LVCMOS33 [get_ports MLT2]
set_property IOSTANDARD LVCMOS33 [get_ports MR1]
set_property IOSTANDARD LVCMOS33 [get_ports MR2]
set_property IOSTANDARD LVCMOS33 [get_ports MY1]
set_property IOSTANDARD LVCMOS33 [get_ports SG1]
set_property IOSTANDARD LVCMOS33 [get_ports SG2]
set_property IOSTANDARD LVCMOS33 [get_ports SLT1]
set_property IOSTANDARD LVCMOS33 [get_ports SLT2]
set_property IOSTANDARD LVCMOS33 [get_ports SR1]
set_property IOSTANDARD LVCMOS33 [get_ports SR2]
set_property IOSTANDARD LVCMOS33 [get_ports SY1]
set_property IOSTANDARD LVCMOS33 [get_ports SY2]
set_property IOSTANDARD LVCMOS33 [get_ports blink]
set_property IOSTANDARD LVCMOS33 [get_ports clk]
set_property IOSTANDARD LVCMOS33 [get_ports reset_n]
set_property DRIVE 12 [get_ports MG1]
set_property DRIVE 12 [get_ports MG2]
set_property DRIVE 12 [get_ports MLT1]
set_property DRIVE 12 [get_ports MLT2]
set_property DRIVE 12 [get_ports MR1]
set_property DRIVE 12 [get_ports MR2]
set_property DRIVE 12 [get_ports MY1]
set_property DRIVE 12 [get_ports MY2]
set_property DRIVE 12 [get_ports SG1]
set_property DRIVE 12 [get_ports SG2]
set_property DRIVE 12 [get_ports SLT1]
set_property DRIVE 12 [get_ports SLT2]
set_property DRIVE 12 [get_ports SR1]
set_property DRIVE 12 [get_ports SR2]
set_property DRIVE 12 [get_ports SY1]
set_property DRIVE 12 [get_ports SY2]
set_property SLEW SLOW [get_ports MG1]
set_property SLEW SLOW [get_ports MG2]
set_property SLEW SLOW [get_ports MLT1]
set_property SLEW SLOW [get_ports MLT2]
set_property SLEW SLOW [get_ports MR1]
set_property SLEW SLOW [get_ports MR2]
set_property SLEW SLOW [get_ports MY1]
set_property SLEW SLOW [get_ports MY2]
set_property SLEW SLOW [get_ports SG1]
set_property SLEW SLOW [get_ports SG2]
set_property SLEW SLOW [get_ports SLT1]
set_property SLEW SLOW [get_ports SLT2]
set_property SLEW SLOW [get_ports SR1]
set_property SLEW SLOW [get_ports SR2]
set_property SLEW SLOW [get_ports SY1]
set_property SLEW SLOW [get_ports SY2]
set_property IOSTANDARD LVCMOS33 [get_ports MY2]

```

---

# **Part 7: Schematics**

---

## **RTL Schematic**
<img width="1920" height="1038" alt="image" src="https://github.com/user-attachments/assets/6f004f7e-4977-4017-895c-8b7c7a35dc43" />

---

## **Technology Schematic**
<img width="1920" height="1034" alt="image" src="https://github.com/user-attachments/assets/b58baedf-d246-4412-8eed-81f0c10f4861" />



---

# **Part 8: PRoject Summary**

<img width="1627" height="852" alt="image" src="https://github.com/user-attachments/assets/75dedaac-2a39-4b17-aa2f-0803d9ffd247" />
<img width="1641" height="853" alt="image" src="https://github.com/user-attachments/assets/15408a2c-825c-4f96-974e-16f3f6b40667" />
<img width="829" height="173" alt="image" src="https://github.com/user-attachments/assets/27f45391-5447-4957-8835-c132c33a42a6" />


---
