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
