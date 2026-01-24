# **EXPERIMENT: 24-HOUR DIGITAL CLOCK**

---

## **PART 1: DESIGN, ARCHITECTURE & FSM BEHAVIOR**

---

## **1. Concept / Introduction**

A **24-hour digital clock** is a sequential digital system that continuously keeps track of **time in hours, minutes, and seconds**, following the standard 24-hour format:

[
00{:}00{:}00 ;\rightarrow; 23{:}59{:}59 ;\rightarrow; 00{:}00{:}00
]

The clock updates its time **once every second**, making it a **time-driven synchronous system**.
Unlike combinational circuits, the output depends on **previous time values**, requiring storage and clocked operation.

---

## **2. Need for Sequential Operation**

A digital clock must remember and update time continuously, which introduces **state dependency**:

* Seconds depend on previous seconds
* Minutes increment only when seconds roll over
* Hours increment only when minutes roll over
* Correct operation requires **memory and clock synchronization**

Therefore, a digital clock **must be implemented as a sequential circuit** using registers and counters.

---

## **3. Role of FSM (Implicit FSM)**

Although the design does not use a large explicit FSM with many named states, it still behaves as an **FSM driven by counter rollovers**.

This is known as a **counter-controlled FSM**.

### FSM Interpretation

* Each valid time (HH:MM:SS) represents a **state**
* Transitions occur on:

  * `seconds == 59`
  * `minutes == 59`
  * `hours == 23`
* All transitions occur **synchronously on clock edges**

This implicit FSM ensures **deterministic and safe time progression**.

---

## **4. Design Strategy**

The clock is designed using a **hierarchical counter approach**:

* Generate a **1-second time base**
* Use cascading counters:

  * Seconds → Minutes → Hours
* Use rollover conditions instead of explicit FSM states
* Prefer **Moore-style behavior** (outputs depend only on registers)

This strategy ensures:

* Glitch-free outputs
* Simple logic
* Easy verification
* FPGA-friendly synthesis

---

## **5. Inputs and Outputs**

### **Top-Level Inputs**

| Signal  | Description                       |
| ------- | --------------------------------- |
| `clk`   | System clock (e.g., 50 MHz)       |
| `reset` | Synchronous or asynchronous reset |

---

### **Top-Level Outputs**

| Signal          | Description         |
| --------------- | ------------------- |
| `hours [4:0]`   | Hour value (0–23)   |
| `minutes [5:0]` | Minute value (0–59) |
| `seconds [5:0]` | Second value (0–59) |

---

## **6. Datapath Architecture**

The datapath is responsible for **time storage and counting**.

### **6.1 Time Base Generation**

In real hardware:

* System clock = **50 MHz**
* Clock period = **20 ns**
* A divider generates **1 Hz** (1 second tick)

[
50{,}000{,}000 \text{ cycles} = 1 \text{ second}
]

This 1-second pulse drives all time updates.

---

### **6.2 Time Counters**

The datapath contains three primary counters:

| Counter         | Range | Purpose           |
| --------------- | ----- | ----------------- |
| Seconds counter | 0–59  | Fine-grain time   |
| Minutes counter | 0–59  | Medium-grain time |
| Hours counter   | 0–23  | Coarse-grain time |

Each counter is implemented using **registers and increment logic**.

---

### **6.3 Cascaded Counter Operation**

* Seconds increment every 1 second
* When `seconds == 59`:

  * Seconds reset to 0
  * Minutes increment
* When `minutes == 59`:

  * Minutes reset to 0
  * Hours increment
* When `hours == 23`:

  * Hours reset to 0

This cascading behavior forms the **core datapath algorithm**.

---

## **7. Control Path Architecture**

The control path does not use a separate controller FSM.
Instead, it relies on **combinational rollover conditions**.

### Control Signals (Conceptual)

* `sec_rollover = (seconds == 59)`
* `min_rollover = (minutes == 59)`
* `hour_rollover = (hours == 23)`

### Control Responsibilities

* Enable minutes counter on `sec_rollover`
* Enable hours counter on `min_rollover`
* Reset hours on `hour_rollover`
* Handle global reset

This approach minimizes hardware while maintaining correctness.

---

## **8. FSM / State Behavior (Rollover-Based FSM)**

### FSM View of Time Progression

| Condition       | Action                     |
| --------------- | -------------------------- |
| Normal tick     | `seconds++`                |
| `seconds == 59` | `seconds = 0`, `minutes++` |
| `minutes == 59` | `minutes = 0`, `hours++`   |
| `hours == 23`   | `hours = 0`                |

All transitions occur **on the rising edge of the clock**, ensuring synchronous behavior.

---

## **9. Simulation Timing Consideration**

For simulation:

* Real-time seconds are **scaled**
* Example:

  * 1 second → 1 ns (or 10 ns)
* Only the **time scale changes**
* Logical behavior remains identical

This scaling:

* Prevents long simulation runtimes
* Does **not affect hardware correctness**

---

## **10. Summary**

* The 24-hour digital clock is a **counter-driven sequential system**
* FSM behavior is **implicit via rollover logic**
* Datapath handles counting and storage
* Control path handles enable and reset logic
* Design is:

  * Simple
  * Deterministic
  * Synthesizable
  * Easily extendable to 12-hour mode

---

# **PART 2: SYNTHESIZABLE VERILOG RTL**

---

## **2.1 Design Requirements (RTL Perspective)**

The RTL design must:

* Generate a **1-second time tick** from a high-frequency clock
* Maintain valid ranges:

  * Seconds: `0–59`
  * Minutes: `0–59`
  * Hours: `0–23`
* Update time synchronously
* Reset cleanly to `00:00:00`
* Avoid latches, delays, or race conditions

---

## **2.2 Top-Level Module Interface**

### **Module Name**

```verilog
digital_clock_24hr
```

### **Inputs**

| Signal  | Purpose           |
| ------- | ----------------- |
| `clk`   | System clock      |
| `reset` | Active-high reset |

### **Outputs**

| Signal          | Purpose             |
| --------------- | ------------------- |
| `hours [4:0]`   | Hour value (0–23)   |
| `minutes [5:0]` | Minute value (0–59) |
| `seconds [5:0]` | Second value (0–59) |

---

## **2.3 RTL Implementation**

```verilog
`timescale 1ns / 1ps
/*---------------------------------------------------------
  24-Hour Digital Clock
  - Counter-driven sequential design
  - Implicit FSM via rollover conditions
---------------------------------------------------------*/

module digital_clock_24hr (
    input  wire clk,
    input  wire reset,
    output reg  [4:0] hours,
    output reg  [5:0] minutes,
    output reg  [5:0] seconds
);

/*---------------------------------------------------------
  Parameters
---------------------------------------------------------*/
/* 
 For hardware:
   50 MHz clock -> 50,000,000 cycles = 1 second

 For simulation:
   This value can be reduced (e.g., 10, 100)
*/
parameter ONE_SEC_COUNT = 50_000_000;

/*---------------------------------------------------------
  1-Second Time Base Generator
---------------------------------------------------------*/
reg [25:0] sec_counter;

wire one_sec_tick;

assign one_sec_tick = (sec_counter == ONE_SEC_COUNT - 1);

always @(posedge clk) begin
    if (reset)
        sec_counter <= 26'd0;
    else if (one_sec_tick)
        sec_counter <= 26'd0;
    else
        sec_counter <= sec_counter + 1'b1;
end

/*---------------------------------------------------------
  Time Counters: Seconds, Minutes, Hours
---------------------------------------------------------*/
always @(posedge clk) begin
    if (reset) begin
        seconds <= 6'd0;
        minutes <= 6'd0;
        hours   <= 5'd0;
    end
    else if (one_sec_tick) begin

        /* Seconds Counter */
        if (seconds == 6'd59) begin
            seconds <= 6'd0;

            /* Minutes Counter */
            if (minutes == 6'd59) begin
                minutes <= 6'd0;

                /* Hours Counter */
                if (hours == 5'd23)
                    hours <= 5'd0;
                else
                    hours <= hours + 1'b1;

            end
            else begin
                minutes <= minutes + 1'b1;
            end

        end
        else begin
            seconds <= seconds + 1'b1;
        end

    end
end

endmodule
```

---

## **2.4 RTL Coding Style and Design Notes**

### **Clocking**

* Single clock domain
* All registers triggered on `posedge clk`

### **Reset**

* Synchronous reset
* Clears:

  * Time counters
  * Second divider counter

### **FSM Style**

* No explicit FSM
* FSM behavior achieved using **rollover conditions**
* Equivalent to a Moore machine

### **Synthesizability**

* No `#delay`
* No blocking assignments in sequential logic
* No latch inference
* Safe for FPGA and ASIC flows

---

## **2.5 Design Characteristics**

* Deterministic behavior
* Glitch-free outputs
* Easy simulation scaling via parameter
* Directly extensible to:

  * 12-hour clock
  * Alarm clock
  * Stopwatch
  * Timer

---



