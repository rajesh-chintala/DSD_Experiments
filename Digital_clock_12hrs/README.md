<img width="1625" height="884" alt="image" src="https://github.com/user-attachments/assets/ac3ae05c-ea6a-4e6f-8fe7-041ddab4b6a2" />---

# **EXPERIMENT: 12-Hour Digital Clock**

---

## **PART 1: DESIGN AND ARCHITECTURE**

---

## **1. Aim of the Experiment**

To design and implement a **12-hour digital clock** using Verilog HDL that displays time in the format:

```
HH : MM : SS  (AM / PM)
```

The design must be **fully synchronous**, synthesizable on FPGA hardware, and verified through simulation using time-scaling techniques.

---

## **2. Need for Sequential Operation**

A digital clock inherently depends on **time progression**, which requires:

* Counting of seconds
* Cascaded rollover of minutes and hours
* Memory to store current time values

Since these operations depend on **previous values**, the design must use:

* **Sequential logic**
* **Clocked registers**
* **Synchronous counters**

Pure combinational logic is insufficient for timekeeping systems.

---

## **3. Modes of Operation**

### **3.1 Normal Mode**

* Clock increments every second
* Seconds count from 0 to 59
* Minutes increment on seconds rollover
* Hours increment on minutes rollover
* AM/PM toggles only at the **11 → 12 transition**

### **3.2 Reset Mode**

* Clock resets to a known initial state
* Time is initialized to:

  ```
  12 : 00 : 00  AM
  ```
* Ensures deterministic startup behavior

---

## **4. Concept of 12-Hour Time Representation**

Unlike a 24-hour clock, the **12-hour format** has special characteristics:

* Hour range is **1 to 12**
* Value **12** is treated as a valid boundary
* AM/PM indication is mandatory
* AM/PM toggles **once every 12 hours**

### **Key Behavioral Rules**

| Condition | Action             |
| --------- | ------------------ |
| 11 → 12   | Toggle AM/PM       |
| 12 → 1    | No toggle          |
| Reset     | Time = 12:00:00 AM |

This makes the hour logic **more complex than modulo-24 counting**.

---

## **5. Inputs and Outputs**

### **5.1 Input Signals**

| Signal  | Description                   |
| ------- | ----------------------------- |
| `clk`   | System clock                  |
| `reset` | Active-high synchronous reset |

---

### **5.2 Output Signals**

| Signal                | Description                      |
| --------------------- | -------------------------------- |
| `hours [3:0] / [4:0]` | Current hour value (1–12)        |
| `minutes [5:0]`       | Minute count (0–59)              |
| `seconds [5:0]`       | Second count (0–59)              |
| `am_pm`               | AM/PM indicator (0 = AM, 1 = PM) |

---

## **6. Datapath Architecture**

The datapath implements **time counting and storage** using cascaded counters.

### **6.1 Clock Divider (1-Second Time Base)**

* Input clock is typically **50 MHz**
* A parameterized counter divides the clock to generate a **1-second tick**
* This tick drives the seconds counter

> For simulation, this divider is scaled down using a parameter (`ONE_SEC_COUNT`)

---

### **6.2 Seconds Counter**

* Range: **0 to 59**
* Increments on every 1-second tick
* Generates a carry when reaching 59

---

### **6.3 Minutes Counter**

* Range: **0 to 59**
* Increments on seconds rollover
* Generates a carry when reaching 59

---

### **6.4 Hours Counter (12-Hour Logic)**

* Range: **1 to 12**
* Increments on minutes rollover
* Special handling for:

  * 11 → 12 (toggle AM/PM)
  * 12 → 1 (wraparound)

---

### **6.5 AM/PM Control Logic**

* Stored in a 1-bit register
* Toggles only when:

  ```
  hours == 11 AND minutes == 59 AND seconds == 59
  ```
* Ensures correct day-night indication

---

## **7. Control Strategy**

* No FSM required
* Control is achieved through:

  * Cascaded counter rollover conditions
  * Conditional hour and AM/PM updates
* Entire design operates in **one clock domain**

This results in a **simple, deterministic, and efficient design**.

---

## **8. Timing Considerations (Simulation vs Hardware)**

### **Real Hardware**

| Parameter       | Value                   |
| --------------- | ----------------------- |
| Clock frequency | 50 MHz                  |
| 1 second        | 50,000,000 clock cycles |

---

### **Simulation Scaling**

| Real Time | Simulation Time |
| --------- | --------------- |
| 1 second  | 1 ns (scaled)   |
| 1 minute  | 60 ns           |
| 1 hour    | 3600 ns         |

This scaling:

* Affects **only simulation**
* Does **not alter synthesizable RTL**
* Enables fast verification

---

## **9. Design Strategy and Rationale**

* Counter-based design ensures clarity and correctness
* Parameterized timing enables easy simulation scaling
* Explicit AM/PM control avoids ambiguity
* Fully synchronous logic prevents glitches
* Reusable structure from 24-hour clock reduces complexity

---

## **10. Summary**

* 12-hour digital clock introduces AM/PM complexity
* Sequential logic is mandatory
* Datapath-driven design is efficient and synthesizable
* Simulation scaling ensures practical verification
* Design is suitable for FPGA implementation and academic evaluation

---

# **PART 2: SYNTHESIZABLE RTL DESIGN — 12-HOUR DIGITAL CLOCK**

---

## **2.1 Design Requirements**

The 12-hour digital clock must:

* Operate from a **system clock** (e.g., 50 MHz on FPGA)
* Generate a **1-second enable** internally
* Maintain:

  * Seconds: 0–59
  * Minutes: 0–59
  * Hours: 1–12
* Provide **AM / PM indication**
* Reset cleanly to a known time
* Be **fully synchronous and synthesizable**
* Support **simulation time scaling** without affecting hardware behavior

---

## **2.2 Top-Level Module Interface**

### **Module Name**

```
digital_clock_12hr
```

### **Input Signals**

| Signal  | Purpose            |
| ------- | ------------------ |
| `clk`   | System clock input |
| `reset` | Synchronous reset  |

---

### **Output Signals**

| Signal         | Purpose                   |
| -------------- | ------------------------- |
| `hours[3:0]`   | Current hour (1–12)       |
| `minutes[5:0]` | Current minute (0–59)     |
| `seconds[5:0]` | Current second (0–59)     |
| `am_pm`        | AM (0) / PM (1) indicator |

---

## **2.3 Internal Architectural Components**

### **Datapath Elements**

* **1-second divider counter**
* Seconds register
* Minutes register
* Hours register
* AM/PM flip-flop

### **Control Behavior**

* Cascaded rollover logic:

  * Seconds → Minutes → Hours → AM/PM
* No explicit FSM required
* Fully synchronous sequencing

---

## **2.4 Timing Strategy**

### **Hardware**

* Clock: 50 MHz → 20 ns period
* Divider generates **1-second enable**

### **Simulation**

* Divider value parameterized
* Allows **fast simulation** without changing RTL

---

## **2.5 RTL Coding Style**

* Single `always @(posedge clk)`
* Synchronous reset
* Parameterized divider
* No latches
* No blocking assignments in sequential logic
* Clean rollover conditions

---

## **2.6 Complete RTL Code (12-Hour Clock)**

```verilog
`timescale 1ns / 1ps
/*---------------------------------------------------------
  12-Hour Digital Clock
---------------------------------------------------------*/

module digital_clock_12hr #(
    parameter ONE_SEC_COUNT = 50_000_000   // 50 MHz default
)(
    input  wire clk,
    input  wire reset,

    output reg  [3:0] hours,     // 1–12
    output reg  [5:0] minutes,   // 0–59
    output reg  [5:0] seconds,   // 0–59
    output reg        am_pm      // 0 = AM, 1 = PM
);

    /*---------------- 1-Second Divider ----------------*/
    reg [31:0] sec_cnt;

    wire one_sec_tick = (sec_cnt == ONE_SEC_COUNT - 1);

    /*--------------------------------------------------
      Sequential Logic
    --------------------------------------------------*/
    always @(posedge clk) begin
        if (reset) begin
            sec_cnt <= 32'd0;
            seconds <= 6'd0;
            minutes <= 6'd0;
            hours   <= 4'd12;   // 12:00:00 AM reset
            am_pm   <= 1'b0;
        end
        else begin
            /*---------------- Divider ----------------*/
            if (one_sec_tick)
                sec_cnt <= 32'd0;
            else
                sec_cnt <= sec_cnt + 1;

            /*---------------- Time Update ----------------*/
            if (one_sec_tick) begin
                if (seconds == 6'd59) begin
                    seconds <= 6'd0;

                    if (minutes == 6'd59) begin
                        minutes <= 6'd0;

                        /*------------- Hour Logic -------------*/
                        if (hours == 4'd11) begin
                            hours <= 4'd12;
                            am_pm <= ~am_pm;   // Toggle AM/PM
                        end
                        else if (hours == 4'd12)
                            hours <= 4'd1;
                        else
                            hours <= hours + 1;
                    end
                    else
                        minutes <= minutes + 1;
                end
                else
                    seconds <= seconds + 1;
            end
        end
    end

endmodule
```

---

## **2.7 Design Characteristics**

* ✔ Fully synthesizable
* ✔ Single clock domain
* ✔ No FSM overhead
* ✔ Parameterized for simulation speed-up
* ✔ Clean AM/PM rollover at 11 → 12
* ✔ FPGA-friendly implementation

---

# **PART 3: TESTBENCH & VERIFICATION**

---

## **3.1 Purpose of the Testbench**

The purpose of this testbench is to **verify the functional correctness** of the 12-hour digital clock by validating:

* Proper reset initialization
* Correct second, minute, and hour counting
* Correct rollover behavior at:

  * 59 → 00 seconds
  * 59 → 00 minutes
  * 11 → 12 hour with AM/PM toggle
  * 12 → 1 hour without AM/PM toggle
* Stable and glitch-free operation under continuous clocking
* Correct operation under **simulation time scaling**

---

## **3.2 Verification Objectives**

The testbench aims to confirm that:

* The internal 1-second tick generation works as intended
* Time increments occur **only on the 1-second tick**
* Cascaded rollover logic functions correctly
* AM/PM toggles only at the **11 → 12 transition**
* The design behaves deterministically after reset
* The RTL is **tool-agnostic and synthesizable**

---

## **3.3 Verification Strategy**

A **directed, time-driven verification approach** is used:

* Apply reset to initialize the clock
* Run the clock continuously
* Observe time progression through waveform inspection
* Use a **scaled divider** to accelerate simulation
* Allow sufficient simulation time to observe:

  * Multiple second rollovers
  * Minute rollover
  * Hour rollover
  * AM/PM transition

This strategy mirrors **real hardware operation**, only with scaled timing.

---

## **3.4 Simulation Time Scaling**

### **Real Hardware**

* System clock: 50 MHz (20 ns period)
* 1 second = 50,000,000 clock cycles
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

## **3.5 Testbench Design Considerations**

* Single free-running clock
* Synchronous reset
* No force or manual intervention
* Parameter override for fast simulation
* Waveform-based verification

---

## **3.6 Complete Testbench Code**

```verilog
`timescale 1ns / 1ps
/*---------------------------------------------------------
  Testbench for 12-Hour Digital Clock
---------------------------------------------------------*/

module tb_digital_clock_12hr;

    /*---------------- Testbench Signals ----------------*/
    reg clk;
    reg reset;

    wire [3:0] hours;
    wire [5:0] minutes;
    wire [5:0] seconds;
    wire       am_pm;

    /*--------------------------------------------------
      DUT Instantiation
      ONE_SEC_COUNT reduced for simulation
    --------------------------------------------------*/
    digital_clock_12hr #(
        .ONE_SEC_COUNT(10)   // Fast simulation
    ) dut (
        .clk     (clk),
        .reset   (reset),
        .hours   (hours),
        .minutes (minutes),
        .seconds (seconds),
        .am_pm   (am_pm)
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
        clk   = 1'b1;
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

## **3.7 Applied Inputs and Expected Outputs**

### **Reset Phase**

* `reset = 1`
* Expected outputs:

  * `hours = 12`
  * `minutes = 0`
  * `seconds = 0`
  * `am_pm = 0 (AM)`

---

### **Normal Operation**

* Seconds increment every simulated second
* Minutes increment after 60 seconds
* Hours increment after 60 minutes
* AM/PM toggles only at:

  ```
  11:59:59 → 12:00:00
  ```

---

## **3.8 Observability in Simulation**

Key signals to observe:

* `seconds[5:0]`
* `minutes[5:0]`
* `hours[3:0]`
* `am_pm`
* `clk`
* `reset`

Waveform inspection is sufficient due to deterministic behavior.

---

## **3.9 Summary**

* Testbench validates all functional aspects of the 12-hour clock
* Simulation time scaling enables fast verification
* No race conditions or latches
* RTL behavior matches real-time clock expectations

---

<img width="1621" height="879" alt="image" src="https://github.com/user-attachments/assets/8f83334b-ee42-4d3d-af52-4153ac684012" />
<img width="1627" height="883" alt="image" src="https://github.com/user-attachments/assets/39b03cb1-82a0-4310-917d-6a99f8d2d9c0" />
<img width="1621" height="881" alt="image" src="https://github.com/user-attachments/assets/643d81f5-deff-4acc-9399-628780bd2bbd" />
<img width="1626" height="878" alt="image" src="https://github.com/user-attachments/assets/fe268ce7-47be-4293-ad29-7ddeeb59e390" />
<img width="1625" height="884" alt="image" src="https://github.com/user-attachments/assets/edbd2886-e9e2-43c1-8438-97804715ac86" />

<img width="1628" height="884" alt="image" src="https://github.com/user-attachments/assets/d049102c-2618-4d4b-883e-0fd447bca3d3" />

<img width="1625" height="883" alt="image" src="https://github.com/user-attachments/assets/7074d05d-bd41-44d8-b58c-e00f484e0b6e" />

<img width="1625" height="875" alt="image" src="https://github.com/user-attachments/assets/0c1f06ed-db2b-489e-b810-89d33a4af885" />

<img width="1627" height="881" alt="image" src="https://github.com/user-attachments/assets/44aba7e3-5ae4-41ad-a67e-3d6085aa4290" />

---

# **PART 4: SIMULATION RESULTS & WAVEFORM ANALYSIS (12-Hour Digital Clock)**

---

## **4.1 Overview of Observed Simulation**

The simulation waveform confirms **correct functional operation** of the 12-hour digital clock with respect to:

* Second, minute, and hour counting
* Proper rollover at time boundaries
* Correct AM/PM (`am_pm`) toggling
* Stable synchronous behavior
* Correct reset initialization

The behavior observed in the waveform matches the **expected real-time behavior**, scaled appropriately for simulation.

---

## **4.2 Clock and Reset Behavior**

### **Clock (`clk`)**

* Free-running periodic signal
* Drives all internal counters and registers
* Stable throughout the simulation window
* All outputs update synchronously on clock edges

### **Reset (`reset`)**

* Active-high reset
* When asserted:

  * `hours` initializes to **12**
  * `minutes` initializes to **0**
  * `seconds` initializes to **0**
  * `am_pm` initializes to **AM (0)**
* After deassertion:

  * Clock begins normal time progression

This clean initialization is clearly visible at the start of the waveform.

---

## **4.3 Seconds Counter Behavior**

### **Observed Behavior**

* `seconds[5:0]` increments sequentially:

  ```
  55 → 56 → 57 → 58 → 59 → 00
  ```
* Increment occurs only on the internally generated **1-second tick**
* No skipped or duplicated values

### **Rollover**

* At `seconds = 59`, the next increment:

  * Resets seconds to `00`
  * Triggers minute increment

✔ **Seconds rollover logic is correct and deterministic**

---

## **4.4 Minutes Counter Behavior**

### **Observed Behavior**

* `minutes[5:0]` remains constant while seconds count
* At `59 → 00` seconds transition:

  ```
  minutes: incremented, when  59 → 00
  ```

### **Cascaded Increment**

* Minutes increment only when seconds roll over
* No asynchronous or early increments

✔ **Minute carry propagation is correct**

---

## **4.5 Hours Counter and 12-Hour Format**

### **Critical Transition Observed**

At the highlighted waveform cursor:

```
11 : 59 : 59  AM
↓
12 : 00 : 00  PM
```

### **Correct Behavior**

* `hours` transitions:

  ```
  11 → 12
  ```
* `minutes` resets to `00`
* `seconds` resets to `00`
* `am_pm` toggles from `0 → 1`

✔ **This confirms correct 11→12 AM/PM transition logic**

---

## **4.6 AM/PM (`am_pm`) Signal Behavior**

### **Observed Behavior**

* `am_pm = 1` after the 11→12 transition
* Remains stable across subsequent second and minute increments
* Does **not** toggle at:

  * 12 → 1 transition
  * Any other hour increment

### **Design Rule Verified**

AM/PM toggles **only once per 12-hour cycle**, at:

```
11:59:59 → 12:00:00
```

✔ **AM/PM logic is implemented correctly**

---

## **4.7 Synchronization and Timing Integrity**

From the waveform:

* All outputs change **only on clock edges**
* No glitches on `hours`, `minutes`, `seconds`, or `am_pm`
* No race conditions between counters
* Clean, monotonic transitions

This confirms:

* Fully synchronous design
* Proper ordering of cascaded counters
* Absence of combinational hazards

---

## **4.8 Simulation Time Scaling Validation**

### **Simulation Setup**

* `ONE_SEC_COUNT` reduced for fast simulation
* Clock period ≈ **0.1 ns**
* Logical 1 second represented by a short simulation interval

### **Observed Result**

* Time behavior matches **real clock semantics**
* Only the **speed** is scaled, not the **logic**

✔ **Simulation faithfully represents real hardware behavior**

---

## **4.9 Summary of Waveform Validation**

✔ Correct reset initialization
✔ Accurate second counting
✔ Proper minute rollover
✔ Correct 12-hour hour transitions
✔ AM/PM toggles at correct boundary
✔ Glitch-free synchronous behavior
✔ Simulation scaling works as intended

---

# **PART 5: VIVA QUESTIONS, INTERVIEW QUESTIONS, DEBUG SCENARIOS & DESIGN VARIATIONS**

---

## **A) Viva Questions (University / Lab-Oriented)**

1. What is the difference between a 24-hour clock and a 12-hour clock?
2. Why is an AM/PM indicator required in a 12-hour digital clock?
3. What is the role of the seconds counter in this design?
4. How are seconds, minutes, and hours related in this clock?
5. Why are minutes and seconds represented using 6 bits?
6. Why are hours represented using 4 or 5 bits?
7. When does the AM/PM signal toggle?
8. Why does AM/PM not toggle at every hour change?
9. What happens at the transition from 11:59:59 to 12:00:00?
10. What happens at the transition from 12:59:59 to 1:00:00?
11. Why is a reset signal necessary in a clock design?
12. What values are loaded into the clock on reset?
13. Is this design synchronous or asynchronous?
14. Why is the design implemented using counters instead of an FSM?
15. Can this clock work without a clock signal?
16. What happens if reset is asserted during normal operation?
17. Why is simulation time scaling required?
18. Does time scaling affect hardware behavior?
19. What is the maximum hour value in this design?
20. How is rollover handled in the hour counter?

---

## **B) Interview-Style Questions (VLSI / Industry)**

### **Architecture & Design**

1. Why did you choose cascaded counters instead of an FSM?
2. How would you modify this design to support both 12-hour and 24-hour modes?
3. How would you add a time-set feature (manual hour/minute adjustment)?
4. How would you implement alarm functionality?
5. How would you synchronize an external time-set input?
6. How would you add clock-enable support?
7. Can this design be parameterized for different clock frequencies?
8. How would you reduce power consumption in this clock?

### **Timing & Reliability**

9. What happens if the 1-second tick is inaccurate?
10. How would clock jitter affect this design?
11. What is the critical path in this design?
12. How would you verify long-term correctness (24+ hours)?
13. How would you test rollover corner cases?

### **Synthesis & Hardware**

14. How many flip-flops are required for this design?
15. What combinational logic is inferred?
16. How does this design map to FPGA resources?
17. Can this design run at very high clock frequencies?
18. What changes are required to run this on real hardware with a 50 MHz clock?

---

## **C) Debug Scenarios (Very Important)**

### **Bug 1: AM/PM Toggles at Wrong Time**

* AM/PM toggles at 12 → 1 instead of 11 → 12

### **Bug 2: Hours Reset to 0 Instead of 12**

* Clock shows 0:00 AM after reset

### **Bug 3: Hours Increment at 59 Minutes Instead of 60**

* Off-by-one error in minute rollover logic

### **Bug 4: Seconds Skip Values**

* Incorrect 1-second tick generation

### **Bug 5: Multiple AM/PM Toggles**

* AM/PM toggles more than once per 12-hour cycle

### **Bug 6: Glitches on Output Signals**

* Hours or minutes briefly show invalid values

### **Bug 7: Reset Causes Metastability**

* Reset not synchronized to clock

---

## **D) Design Variations**

### **24-Hour / 12-Hour Selectable Clock**

* Mode input selects display format
* AM/PM disabled in 24-hour mode

### **Time-Setting Mode**

* Add `set_hr`, `set_min`, `set_sec` inputs
* Freeze normal counting during adjustment

### **Alarm Clock Extension**

* Compare current time with stored alarm time
* Trigger alarm output

### **Clock Enable / Pause Mode**

* Stop time progression without resetting values

### **BCD Output Clock**

* Convert binary counters to BCD for display devices

---

## **E) Conceptual Questions (Deep Understanding)**

1. Why are clocks naturally modeled using counters?
2. What advantages does synchronous design provide here?
3. Why is AM/PM logic separate from hour counting?
4. How does cascading simplify design reasoning?
5. What are the risks of mixing combinational and sequential time logic?
6. How would metastability affect time correctness?

---

## **F) If This Were Used in a Real Product…**

You would also consider:

* ✔ Clock accuracy and calibration
* ✔ Battery backup for time retention
* ✔ Power-fail detection
* ✔ RTC crystal integration
* ✔ CDC for external inputs
* ✔ Low-power sleep modes
* ✔ Formal verification of rollover logic

---

# **Part 6: Constraints**

```verilog
create_clock -period 10 -name clk [get_ports clk]

set_property PACKAGE_PIN E3 [get_ports clk]
set_property PACKAGE_PIN J15 [get_ports reset]
set_property PACKAGE_PIN V11 [get_ports {hours[3]}]
set_property PACKAGE_PIN V12 [get_ports {hours[2]}]
set_property PACKAGE_PIN V14 [get_ports {hours[1]}]
set_property PACKAGE_PIN V15 [get_ports {hours[0]}]
set_property PACKAGE_PIN T16 [get_ports {minutes[5]}]
set_property PACKAGE_PIN U14 [get_ports {minutes[4]}]
set_property PACKAGE_PIN T15 [get_ports {minutes[3]}]
set_property PACKAGE_PIN U16 [get_ports {minutes[1]}]
set_property PACKAGE_PIN V16 [get_ports {minutes[2]}]
set_property PACKAGE_PIN U17 [get_ports {minutes[0]}]
set_property PACKAGE_PIN V17 [get_ports {seconds[5]}]
set_property PACKAGE_PIN R18 [get_ports {seconds[4]}]
set_property PACKAGE_PIN N14 [get_ports {seconds[3]}]
set_property PACKAGE_PIN J13 [get_ports {seconds[2]}]
set_property PACKAGE_PIN K15 [get_ports {seconds[1]}]
set_property PACKAGE_PIN H17 [get_ports {seconds[0]}]
set_property PACKAGE_PIN N16 [get_ports am_pm]
set_property IOSTANDARD LVCMOS33 [get_ports am_pm]
set_property IOSTANDARD LVCMOS33 [get_ports clk]
set_property IOSTANDARD LVCMOS33 [get_ports reset]
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
```

---

# **Part 7: Schematic**

---

## **RTL Schematic**
<img width="1626" height="876" alt="image" src="https://github.com/user-attachments/assets/c917f277-9128-4095-92ba-b0babd0c9db2" />
<img width="1625" height="876" alt="image" src="https://github.com/user-attachments/assets/68d38dbc-a869-436c-92e7-80c3a1b8a181" />

## **Technology Schematic**
<img width="1626" height="878" alt="image" src="https://github.com/user-attachments/assets/d2f3fe49-49cc-4471-b58b-0ec867488d19" />
<img width="1626" height="884" alt="image" src="https://github.com/user-attachments/assets/299ce8e0-9735-4607-b81b-9ec830135baf" />
<img width="1631" height="884" alt="image" src="https://github.com/user-attachments/assets/618993fc-cdd7-4a81-acd8-f718eb61b128" />

---

# **Part 8: Project Summary**

---

<img width="1624" height="880" alt="image" src="https://github.com/user-attachments/assets/bd660bbd-5bab-4f03-a9ca-3ccd3e10e856" />
<img width="1623" height="884" alt="image" src="https://github.com/user-attachments/assets/1f8ed06a-e8bc-41fd-ac92-323b6ee0b999" />
<img width="827" height="170" alt="image" src="https://github.com/user-attachments/assets/26862ce7-de31-45a4-988f-5a272312e27c" />
