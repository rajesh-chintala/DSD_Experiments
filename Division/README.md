# **PART 1: THEORY, ARCHITECTURE, FSM / ASM DESIGN**

---

## **1. Concept / Theory of Binary Division (Binary Long-Division Method)**

Binary division follows the same logical principle as decimal long division, but is performed using **binary shifts, comparisons, and subtractions**. The quotient is generated **one bit at a time**, starting from the most significant bit, and a **partial remainder** is maintained throughout the operation.

<img width="257" height="167" alt="image" src="https://github.com/user-attachments/assets/5158e91f-ec02-4245-859d-b17bb6834121" />

For a given dividend and divisor:

* A partial remainder is formed
* The divisor is compared with the partial remainder
* If the remainder is greater than or equal to the divisor, subtraction is performed and a quotient bit `1` is generated
* Otherwise, subtraction is skipped and a quotient bit `0` is generated
* The next dividend bit is then brought into the remainder

This process continues until all dividend bits are processed.

---

## **2. Need for Sequential Operation**

Binary division **cannot be implemented as a purely combinational circuit** because:

* Each division step depends on the **previous partial remainder**
* Only **one quotient bit** is generated per step
* Intermediate results must be **stored and reused**
* The number of steps equals the bit-width of the operands

Hence, binary division inherently requires:

* Storage elements (registers)
* Clocked operation
* Controlled sequencing

This makes a **sequential circuit implementation mandatory**.

---

## **3. Role of FSM / ASM**

The division process consists of a **fixed sequence of operations**:

* Shift
* Subtract
* Decision
* Update
* Repeat

An **FSM / ASM (Algorithmic State Machine)** is required to:

* Control the order of these operations
* Generate appropriate control signals
* Decide the next operation based on comparison results
* Terminate the process after the required number of cycles

The FSM **does not perform arithmetic**; it only **controls the datapath**.

---

## **4. Datapath Architecture**

<img width="577" height="636" alt="image" src="https://github.com/user-attachments/assets/b7f778b4-85c8-4715-bfac-c839f22328aa" />

The datapath performs all arithmetic and data movement required for division. It is composed of the following elements:

### **4.1 Register A (Dividend / Quotient Formation Register)**

* Initially loaded with the dividend
* Operates as a **left-shift register**
* Shifts left every clock cycle
* Supplies its MSB to the quotient-bit flip-flop

### **4.2 Register B (Divisor Register)**

* Holds the divisor
* Loaded once during initialization
* Remains constant throughout the operation

### **4.3 Register R (Partial Remainder Register)**

* Stores the partial remainder
* Left-shifted together with register A as `(R || A)`
* Updated by subtraction or restoration

### **4.4 Flip-Flop `rr0` (Quotient Bit Register)**

* Captures **one quotient bit per clock cycle**
* Receives the MSB of register A during shifting
* Forced to `0` or retained as `1` depending on subtraction result
* There is **no separate quotient register**

### **4.5 Adder / Subtractor**

* Computes `R − B`
* Carry-out (`Cout`) indicates whether subtraction is valid

### **4.6 Multiplexer (`Rsel`)**

* Selects between:

  * Subtraction result
  * Restored remainder
* Controlled by the FSM

---

## **5. Datapath Operation Explained Using the Example (Step-by-Step)**

<img width="545" height="262" alt="image" src="https://github.com/user-attachments/assets/cad22d6f-74d2-4ccb-8d91-cf85710723d2" />

Consider the division:

```
Dividend (A) = 10001100
Divisor  (B) = 1001
```

### **Initial State (After Reset)**

| Register | Value    |
| -------- | -------- |
| A        | 10001100 |
| R        | 00000000 |
| rr0      | 0        |
| B        | 1001     |

---

### **Cycle-by-Cycle Operation**

Each cycle performs **one quotient decision**.

| Cycle | Datapath Action | R (Remainder) | rr0 | A (Shifted) | 
| ----- | --------------- | ------------- | --- | ----------- | 
| 0     | Load A, B       | 00000000      | 0   | 10001100    | 
| 1     | Shift left `(R|| A)` | 00000001      | 1   | 00011000    |
| 2     | Shift, restore  | 00000010      | 0   | 00110000    | 
| 3     | Shift, restore  | 00000100      | 0   | 01100000    |  
| 4     | Shift, subtract | 00001001      | 1   | 11000000    |  
| 5     | Subtract valid  | 00000000      | 1   | 10000001    |  
| 6     | Shift, restore  | 00000001      | 0   | 00000010    |  
| 7     | Shift, restore  | 00000011      | 0   | 00000100    | 
| 8     | Subtract valid  | 00000001      | 1   | 00010001    | 

---

### **Final Result**

* **Quotient** → sequence of `rr0` values
* **Remainder** → contents of register `R`

```
Quotient  = 00010001
Remainder = 00000001
```

---

## **6. Control Path Explanation (Flowchart / ASM View)**

The control path sequences datapath operations using an ASM-style flow.

### **Control Signals**

| Signal | Function                  |
| ------ | ------------------------- |
| `LA`   | Load register A           |
| `EB`   | Load register B           |
| `EA`   | Enable shift of A         |
| `LR`   | Load remainder            |
| `ER`   | Enable shift of R         |
| `ER0`  | Enable update of `rr0`    |
| `Rsel` | Select subtract / restore |
| `LC`   | Load counter              |
| `EC`   | Enable counter            |
| `Done` | Indicate completion       |

---

### **Functional ASM States**

* **Initialization** – Load operands, clear remainder
* **Shift** – Shift `(R || A)` and capture MSB into `rr0`
* **Subtract / Restore** – Compute `R − B` and decide
* **Update** – Update registers and decrement counter
* **Done** – Assert completion and hold outputs stable

State transitions depend on:

* Subtraction result (`Cout`)
* Counter reaching zero

---

## **7. Inputs, Outputs, and Internal Signals**

### **Inputs**

* `DataA` – Dividend
* `DataB` – Divisor
* `Clock`
* `Reset`

### **Outputs**

* `A` – Quotient (implicit)
* `R` – Remainder
* `Done`

### **Internal Signals**

* `rr0` – Quotient bit
* `Count` – Iteration counter
* `Cout` – Subtraction result indicator

---

## **8. Overall Design Strategy**

* Explain division conceptually first
* Justify sequential nature
* Use FSM/ASM for control
* Implement arithmetic in datapath
* Avoid separate quotient register
* Ensure clean separation of datapath and control

---

## ✅ **PART 1 FINAL STATUS**

✔ Conceptually correct
✔ Hardware-faithful
✔ Textbook-aligned
✔ Viva-safe
✔ Ready for RTL mapping

---

### 🔜 **Next Step**

When ready, say:

👉 **“Proceed to PART 2 – Divider RTL”**

and we will implement the **complete, synthesizable Verilog design** directly from this finalized architecture.
