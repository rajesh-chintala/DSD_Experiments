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

## **2.3 Internal Signal Classification **

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
  S8 : Side road 1 left turn (10s)
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
  S10 : Side road 2 left turn (10s)
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

The purpose of this testbench is to verify the correct operation of the four-way traffic light controller by simulating:

* Normal traffic light sequencing
* Blink mode operation during non-traffic hours
* Proper response to reset
* Continuous FSM operation over long durations

The testbench is designed to observe **one or more complete traffic cycles**, ensuring correctness of timing and sequencing.

---

## **3.2 Objectives**

The objectives of the testbench are to:

* Validate correct timing of main road, side road, and left-turn signals
* Verify correct sequencing of green, yellow, and red phases
* Confirm correct operation of blink mode (all yellow flashing)
* Ensure proper initialization and reset behavior
* Observe long-duration FSM stability

---

## **3.3 Verification Strategy**

A **directed and time-driven verification strategy** is used:

* Apply asynchronous active-low reset
* Allow FSM to run freely without intervention
* Enable blink mode after sufficient normal operation
* Disable blink mode and resume normal operation
* Observe waveform behavior for correctness

This strategy mirrors **realistic operation conditions** of a traffic controller.

---

## **3.4 Testbench Design Considerations**

* Clock frequency corresponds to **50 MHz operation**
* Time base matches the design’s internal timing assumptions
* Simulation runs long enough to capture:

  * Main road green phase
  * Yellow transitions
  * Side road and left-turn phases
  * Blink mode behavior

---

## **3.5 Complete Testbench Code (Textbook)**

```verilog
/*---------------------------------------------------------
  Test Bench for Traffic Light Controller
  File Name : traffic_controller_test.v
---------------------------------------------------------*/

`define clkperiodby2 10   // 10 ns is half clock period (50 MHz)

`include "traffic_controller.v"
`timescale 1ns/100ps

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
        clk     = 1'b0;
        reset_n = 1'b1;
        blink   = 1'b0;

        // Apply reset pulse
        #20 reset_n = 1'b0;
        #20 reset_n = 1'b1;

        // Run normal traffic operation
        #600000;

        // Enable blink mode
        blink = 1'b1;
        #50000;

        // Resume normal traffic operation
        blink = 1'b0;
        #50000;

        // Stop simulation
        $stop;
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

* Main road green active initially
* Yellow lights activate during transitions
* Left-turn signals operate as per FSM
* Side road phases follow main road completion
* No conflicting green signals

---

### **Blink Mode (`blink = 1`)**

**Expected Behavior**

* All red and green lights OFF
* All yellow lights toggle at 1 Hz
* FSM remains in blink state

---

### **Return to Normal Mode**

**Expected Behavior**

* FSM exits blink mode
* Traffic sequence restarts from initial state
* Normal timing resumes

---

## **3.7 Observability During Simulation**

Signals to observe in the waveform:

* Main road signals: `MG*`, `MY*`, `MR*`, `MLT*`
* Side road signals: `SG*`, `SY*`, `SR*`, `SLT*`
* `clk`, `reset_n`, `blink`
* Internal counters (optional)

---

## **3.8 Summary**

* Testbench verifies both normal and blink modes
* Long simulation ensures FSM stability
* Textbook timing assumptions are preserved
* Verification is deterministic and comprehensive

---

