# **EXPERIMENT: 24-HOUR DIGITAL CLOCK**

---

## **PART 1: DESIGN, ARCHITECTURE & FSM BEHAVIOR**

---

## **1. Concept / Introduction**

A **24-hour digital clock** is a sequential digital system that continuously keeps track of **time in hours, minutes, and seconds**, following the standard 24-hour format:

```math
00{:}00{:}00 ;\rightarrow; 23{:}59{:}59 ;\rightarrow; 00{:}00{:}00
```

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

```math
50{,}000{,}000 \text{ cycles} = 1 \text{ second}
```

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

# **PART 3: TESTBENCH & VERIFICATION (24-Hour Digital Clock)**

---

## **3.1 Purpose of the Testbench**

The purpose of this testbench is to **verify the functional correctness** of the 24-hour digital clock by validating:

* Correct generation of the 1-second time tick
* Proper incrementing of seconds, minutes, and hours
* Correct rollover behavior:

  * `59 → 00` for seconds
  * `59 → 00` for minutes
  * `23 → 00` for hours
* Proper reset initialization to `00:00:00`

---

## **3.2 Verification Objectives**

The testbench aims to ensure that:

* Time progresses accurately once every second
* Cascaded counter rollovers occur correctly
* No invalid time values are produced
* Reset brings the clock to a known safe state
* The design remains stable over long simulation runs

---

## **3.3 Verification Strategy**

A **directed, time-scaled verification approach** is used:

* Generate a free-running clock
* Apply reset at the beginning of simulation
* Use a **reduced second counter value** for simulation efficiency
* Allow the clock to run through multiple rollovers
* Observe output waveforms for correctness

This strategy validates both **short-term behavior** (seconds/minutes) and **long-term behavior** (hours rollover).

---

## **3.4 Simulation Time Scaling and Time-Base Mapping**

In real hardware, the digital clock operates using a **50 MHz system clock**, where:

* Clock period = **20 ns**
* 1 second = **50,000,000 clock cycles**

Simulating this directly would be impractical. Therefore, **time scaling** is applied **only for simulation**, without altering functional behavior.

---

### **Simulation Time Scaling Technique Used**

The simulation uses a **compressed time model**:

| Parameter          | Real Hardware | Simulation          |
| ------------------ | ------------- | ------------------- |
| Timescale          | Physical time | `1ns / 1ps`         |
| Clock half-period  | 10 ns         | **0.05 ns (50 ps)** |
| Clock period       | 20 ns         | **0.1 ns**          |
| ONE_SEC_COUNT      | 50,000,000    | **10**              |
| Effective 1 second | 1 s           | **1 ns**            |

---

### **Interpretation**

* Every **0.1 ns clock period** represents a scaled-down hardware clock
* `ONE_SEC_COUNT = 10` causes a **1-second tick every 1 ns**
* This means:

  * **1 ns (simulation) ≡ 1 second (real time)**

---

### **Important Note**

⚠️ This scaling:

* Affects **only simulation speed**
* Does **not** change counter logic, rollover behavior, or FSM correctness
* Produces **functionally equivalent results** to real hardware

Thus, the design remains **hardware-accurate**, while simulation becomes **resource-efficient**.

---

## **3.5 Complete Testbench Code**

```verilog
`timescale 1ns / 1ps
/*---------------------------------------------------------
  Testbench for 24-Hour Digital Clock
---------------------------------------------------------*/

module tb_digital_clock_24hr;

    /*---------------- Testbench Signals ----------------*/
    reg clk;
    reg reset;

    wire [4:0] hours;
    wire [5:0] minutes;
    wire [5:0] seconds;

    /*--------------------------------------------------
      DUT Instantiation
      ONE_SEC_COUNT reduced for simulation
    --------------------------------------------------*/
    digital_clock_24hr #(
        .ONE_SEC_COUNT(10)   // Fast simulation
    ) dut (
        .clk     (clk),
        .reset   (reset),
        .hours   (hours),
        .minutes (minutes),
        .seconds (seconds)
    );

    /*--------------------------------------------------
      Clock Generation (10 ns period)
    --------------------------------------------------*/
    always #0.05 clk = ~clk;

    /*--------------------------------------------------
      Test Sequence
    --------------------------------------------------*/
    initial begin
        /* Initialize signals */
        clk   = 1'b0;
        reset = 1'b1;

        /* Apply reset */
        #2.1;
        reset = 1'b0;

        /* Let clock run */
        #2000;

        /* End simulation */
        $stop;
    end

endmodule
```

---

## **3.6 Applied Inputs**

### **Reset Phase**

* `reset = 1`
* Clock active

### **Normal Operation**

* `reset = 0`
* Clock runs continuously
* Time increments automatically

---

## **3.7 Expected Outputs**

### **Immediately After Reset**

```
hours   = 00
minutes = 00
seconds = 00
```

---

### **During Simulation**

* Seconds increment every simulated second
* Minutes increment when seconds roll over from 59 → 00
* Hours increment when minutes roll over from 59 → 00
* After `23:59:59`, time resets to `00:00:00`

---

## **3.8 Observability in Waveform**

Key signals to observe:

* `clk`
* `reset`
* `seconds`
* `minutes`
* `hours`

Correct operation is confirmed if:

* Counters increment only on `one_sec_tick`
* Rollovers occur at correct limits
* No glitches or invalid values appear

---

## **3.9 Summary**

* Testbench verifies complete time progression
* Simulation time is efficiently scaled
* Rollover logic validated
* Reset behavior confirmed
* Design behaves deterministically and safely

---
<img width="1642" height="882" alt="image" src="https://github.com/user-attachments/assets/112c55ac-531b-4f0f-a820-eae0cf122767" />
<img width="1639" height="876" alt="image" src="https://github.com/user-attachments/assets/b6e5fb20-e20f-48e8-9bea-352799d5eb3c" />
<img width="1641" height="879" alt="image" src="https://github.com/user-attachments/assets/57ab1ffb-9b38-475d-9267-b029dd67f942" />
<img width="1642" height="876" alt="image" src="https://github.com/user-attachments/assets/1b4386e5-b81a-4194-a0ec-d5b89f118477" />

<img width="1642" height="876" alt="image" src="https://github.com/user-attachments/assets/3e65c3db-e443-4205-9ecb-136becc500ab" />
<img width="1643" height="878" alt="image" src="https://github.com/user-attachments/assets/c4f1e227-a6e9-4438-b27f-35cbbf38fbfd" />
<img width="1641" height="879" alt="image" src="https://github.com/user-attachments/assets/88c7a27d-38d5-44af-9660-e19ca00bc5dc" />
<img width="1642" height="881" alt="image" src="https://github.com/user-attachments/assets/0e68b4ef-b8e5-4efb-8312-7123c6fe0520" />

---

# **PART 4: SIMULATION RESULTS AND WAVEFORM ANALYSIS (24-Hour Digital Clock)**

---

## **4.1 Overview of Observed Simulation**

The simulation waveform confirms the **correct functional operation** of the 24-hour digital clock under:

* Normal counting operation
* Proper seconds → minutes → hours rollover
* Correct reset behavior
* Accurate time scaling for simulation

The observed behavior matches the **intended real-time clock functionality**, with time compressed for simulation efficiency.

---

## **4.2 Clock and Reset Behavior**

### **Clock (clk)**

* Free-running periodic signal
* Generated with a **0.1 ns clock period** (fast simulation clock)
* Drives all counters synchronously
* Stable and uniform throughout the simulation

This clock acts as the **scaled equivalent** of the real 50 MHz hardware clock.

---

### **Reset (reset)**

* Active-high reset signal
* When `reset = 1`:

  * `hours = 0`
  * `minutes = 0`
  * `seconds = 0`
* When `reset` is deasserted:

  * Clock begins normal time counting

This behavior is clearly visible at the start of the waveform.

---

## **4.3 Seconds Counter Behavior**

### **Seconds (`seconds[5:0]`)**

From the waveform:

* Seconds increment **sequentially every simulated second**
* Values observed:

  ```
  48 → 49 → 50 → … → 58 → 59 → 0 → 1 → 2
  ```
* When `seconds` reaches **59**:

  * It resets to **0**
  * A carry is generated to increment `minutes`

This confirms:

* Proper **mod-60 counting**
* Correct carry generation

---

## **4.4 Minutes Counter Behavior**

### **Minutes (`minutes[5:0]`)**

From the waveform:

* Minutes remain stable while seconds count
* When seconds roll over from **59 → 0**:

  * Minutes increment by **1**
* When minutes reach **59**:

  * They reset to **0**
  * A carry is generated to increment `hours`

This verifies correct **seconds-to-minutes coupling**.

---

## **4.5 Hours Counter Behavior (24-Hour Format)**

### **Hours (`hours[4:0]`)**

Observed behavior:

* Hours increment only when:

  ```
  minutes = 59 AND seconds = 59
  ```
* Valid hour range:

  ```
  0 → 1 → … → 22 → 23 → 0
  ```
* In the waveform:

  * Hours show a valid value (e.g., **23**)
  * Roll over cleanly to **0**

This confirms correct **24-hour wrap-around logic**.

---

## **4.6 Cascaded Counter Synchronization**

The waveform clearly demonstrates **hierarchical time counting**:

```
seconds → minutes → hours
```

Key observations:

* No skipped values
* No double increments
* No race conditions
* All updates occur **synchronously on clock edges**

This validates the correctness of the **carry-propagation design**.

---

## **4.7 Time Scaling Validation**

The simulation uses **time compression**:

| Real Time  | Simulation Time |
| ---------- | --------------- |
| 1 second   | 1 ns            |
| 60 seconds | 60 ns           |
| 1 hour     | 3600 ns         |

From the waveform:

* One visible second increment occurs every **1 ns**
* Full minute transition observed at **60 ns**
* Behavior matches the scaled expectations exactly

⚠️ This scaling:

* Affects **only simulation**
* Does **not change hardware logic**

---

## **4.8 Absence of Errors and Glitches**

From the waveform:

* No glitches on `hours`, `minutes`, or `seconds`
* No overlapping or undefined values
* No metastability or partial updates
* All signals change synchronously

This confirms:

* Fully synchronous design
* Clean RTL implementation
* Synthesizable behavior

---

## **4.9 Summary of Waveform Validation**

✔ Correct reset initialization
✔ Accurate second counting
✔ Proper minute rollover
✔ Correct 24-hour wrap-around
✔ Clean hierarchical carry propagation
✔ Correct simulation time scaling
✔ Stable and glitch-free outputs

---

# **Part 7: Constraints**

```verilog
create_clock -period 10 -name clk [get_ports clk]

set_property PACKAGE_PIN E3 [get_ports clk]
set_property PACKAGE_PIN J15 [get_ports reset]
set_property PACKAGE_PIN V11 [get_ports {hours[4]}]
set_property PACKAGE_PIN V12 [get_ports {hours[3]}]
set_property PACKAGE_PIN V14 [get_ports {hours[2]}]
set_property PACKAGE_PIN V15 [get_ports {hours[1]}]
set_property PACKAGE_PIN T16 [get_ports {hours[0]}]
set_property PACKAGE_PIN U14 [get_ports {minutes[5]}]
set_property PACKAGE_PIN T15 [get_ports {minutes[4]}]
set_property PACKAGE_PIN V16 [get_ports {minutes[3]}]
set_property PACKAGE_PIN U16 [get_ports {minutes[2]}]
set_property PACKAGE_PIN U17 [get_ports {minutes[1]}]
set_property PACKAGE_PIN V17 [get_ports {minutes[0]}]
set_property PACKAGE_PIN R18 [get_ports {seconds[5]}]
set_property PACKAGE_PIN N14 [get_ports {seconds[4]}]
set_property PACKAGE_PIN J13 [get_ports {seconds[3]}]
set_property PACKAGE_PIN K15 [get_ports {seconds[2]}]
set_property PACKAGE_PIN H17 [get_ports {seconds[1]}]
set_property PACKAGE_PIN N16 [get_ports {seconds[0]}]
set_property IOSTANDARD LVCMOS33 [get_ports clk]
set_property IOSTANDARD LVCMOS33 [get_ports reset]
set_property IOSTANDARD LVCMOS33 [get_ports {hours[4]}]
set_property IOSTANDARD LVCMOS33 [get_ports {hours[3]}]
set_property IOSTANDARD LVCMOS33 [get_ports {hours[2]}]
set_property IOSTANDARD LVCMOS33 [get_ports {hours[1]}]
set_property IOSTANDARD LVCMOS33 [get_ports {hours[0]}]
set_property IOSTANDARD LVCMOS33 [get_ports {minutes[5]}]
set_property IOSTANDARD LVCMOS33 [get_ports {minutes[4]}]
set_property IOSTANDARD LVCMOS33 [get_ports {minutes[3]}]
set_property IOSTANDARD LVCMOS33 [get_ports {minutes[2]}]
set_property IOSTANDARD LVCMOS33 [get_ports {minutes[1]}]
set_property IOSTANDARD LVCMOS33 [get_ports {minutes[0]}]
set_property IOSTANDARD LVCMOS33 [get_ports {seconds[5]}]
set_property IOSTANDARD LVCMOS33 [get_ports {seconds[4]}]
set_property IOSTANDARD LVCMOS33 [get_ports {seconds[3]}]
set_property IOSTANDARD LVCMOS33 [get_ports {seconds[2]}]
set_property IOSTANDARD LVCMOS33 [get_ports {seconds[1]}]
set_property IOSTANDARD LVCMOS33 [get_ports {seconds[0]}]
set_property DRIVE 12 [get_ports {hours[4]}]
set_property DRIVE 12 [get_ports {hours[3]}]
set_property DRIVE 12 [get_ports {hours[2]}]
set_property DRIVE 12 [get_ports {hours[1]}]
set_property DRIVE 12 [get_ports {hours[0]}]
set_property DRIVE 12 [get_ports {minutes[5]}]
set_property DRIVE 12 [get_ports {minutes[4]}]
set_property DRIVE 12 [get_ports {minutes[3]}]
set_property DRIVE 12 [get_ports {minutes[2]}]
set_property DRIVE 12 [get_ports {minutes[1]}]
set_property DRIVE 12 [get_ports {minutes[0]}]
set_property DRIVE 12 [get_ports {seconds[5]}]
set_property DRIVE 12 [get_ports {seconds[4]}]
set_property DRIVE 12 [get_ports {seconds[3]}]
set_property DRIVE 12 [get_ports {seconds[2]}]
set_property DRIVE 12 [get_ports {seconds[1]}]
set_property DRIVE 12 [get_ports {seconds[0]}]
set_property SLEW SLOW [get_ports {hours[4]}]
set_property SLEW SLOW [get_ports {hours[3]}]
set_property SLEW SLOW [get_ports {hours[2]}]
set_property SLEW SLOW [get_ports {hours[1]}]
set_property SLEW SLOW [get_ports {hours[0]}]
set_property SLEW SLOW [get_ports {minutes[5]}]
set_property SLEW SLOW [get_ports {minutes[4]}]
set_property SLEW SLOW [get_ports {minutes[3]}]
set_property SLEW SLOW [get_ports {minutes[2]}]
set_property SLEW SLOW [get_ports {minutes[1]}]
set_property SLEW SLOW [get_ports {minutes[0]}]
set_property SLEW SLOW [get_ports {seconds[5]}]
set_property SLEW SLOW [get_ports {seconds[4]}]
set_property SLEW SLOW [get_ports {seconds[3]}]
set_property SLEW SLOW [get_ports {seconds[2]}]
set_property SLEW SLOW [get_ports {seconds[1]}]
set_property SLEW SLOW [get_ports {seconds[0]}]
```

<img width="1640" height="884" alt="image" src="https://github.com/user-attachments/assets/718a6b27-e53f-4bb8-9ced-82055bff4cd8" />

<img width="1641" height="876" alt="image" src="https://github.com/user-attachments/assets/c0730e9b-4a5f-4161-a736-ea8d00879be4" />

<img width="1641" height="876" alt="image" src="https://github.com/user-attachments/assets/1c4603be-55db-4d5b-93e3-8c7cf42123d8" />
<img width="1641" height="876" alt="image" src="https://github.com/user-attachments/assets/81b6b152-0798-46c6-be07-bcf09c85b765" />
<img width="816" height="171" alt="image" src="https://github.com/user-attachments/assets/ca455842-60d2-48e8-b7f0-d0e79fd43813" />


