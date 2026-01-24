
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
