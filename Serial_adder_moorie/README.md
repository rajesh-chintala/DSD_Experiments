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

## ✅ **PART 1 FINALIZED (ALL REFINEMENTS APPLIED)**

* ✔ Moore vs Mealy justification included
* ✔ Images removed as requested
* ✔ Conceptual clarity preserved
* ✔ State transition table retained
* ✔ Lab-record and viva ready

---

When you are ready, reply:

👉 **“Proceed to PART 2 – Moore Serial Adder RTL”**

and we will move forward with the **Verilog implementation**, clearly showing the **Moore FSM structure and output-from-state logic**.
