# FHE Polynomial Multiplication Optimization

**Project Overview, Research Approach and Implementation Roadmap**

![Scheme](https://img.shields.io/badge/scheme-CKKS-0E7C86)
![Library](https://img.shields.io/badge/library-OpenFHE%201.6.1-0E7C86)
![Hardware](https://img.shields.io/badge/hardware-PolarFire%20SoC%20FPGA-0E7C86)
![Status](https://img.shields.io/badge/status-investigation%20%26%20baseline-F4A261)

> **Status:** Investigation and baseline-establishment stage. No optimization has been finalized or claimed.

---

## Contents

1. [Introduction](#1-introduction)
2. [What Exactly Are We Trying to Improve?](#2-what-exactly-are-we-trying-to-improve)
3. [Why Are We Using OpenFHE?](#3-why-are-we-using-openfhe)
4. [Understanding the Computation](#4-understanding-the-computation)
5. [Our Current Understanding of OpenFHE](#5-our-current-understanding-of-openfhe)
6. [Benchmarking: Establishing a Performance Baseline](#6-benchmarking-establishing-a-performance-baseline)
7. [Proposed Optimization Areas](#7-proposed-optimization-areas)
8. [Hardware, Software and Compiler-Level Approach](#8-hardware-software-and-compiler-level-approach)
9. [Proposed Experimental Methodology](#9-proposed-experimental-methodology)
10. [FPGA-Based Testing Using PolarFire SoC](#10-fpga-based-testing-using-polarfire-soc)
11. [Expected Evaluation](#11-expected-evaluation)
12. [Current Progress and Immediate Next Steps](#12-current-progress-and-immediate-next-steps)
13. [Project Scope and Long-Term Direction](#13-project-scope-and-long-term-direction)

---

## Project at a Glance

| | |
|---|---|
| **The Problem** | FHE lets computers process data that stays encrypted, but the operations are significantly slower than their unencrypted equivalents. |
| **The Target** | Polynomial multiplication inside the **CKKS** scheme, one of its core internal operations, studied inside OpenFHE. |
| **The Method** | Measurement first: trace the code, build a baseline, find the bottleneck, then test one change at a time. |
| **The Extension** | Software optimization first, then FPGA acceleration of selected operations on a PolarFire SoC board. |

> **Central research question:** Can the computational latency of CKKS polynomial multiplication be reduced through improvements in its underlying mathematical operations, software implementation, and hardware-level execution?

---

## 1. Introduction

**Fully Homomorphic Encryption (FHE)** allows computations to be performed directly on encrypted data.

In ordinary encryption, data usually has to be decrypted before it can be processed. FHE changes this: a computer can perform mathematical operations while the information **remains encrypted**.

**Example:** a cloud service needs to calculate something using a user's private information. With FHE, it performs the calculation without directly seeing the original information.

> **The trade-off:** FHE gives an important privacy advantage, but FHE operations can be significantly slower than their ordinary, unencrypted equivalents. This project investigates ways to reduce that computational overhead.

---

## 2. What Exactly Are We Trying to Improve?

The research focuses on the **CKKS** encryption scheme, commonly used for calculations involving approximate numerical values, such as decimal numbers. Within it, we target **polynomial multiplication**, one of the important mathematical operations used internally.

Encrypted calculations are not performed directly on ordinary numbers. Information is represented using mathematical structures called **polynomials**. Multiplying two of them involves several smaller operations:

- Transforming polynomial data into a format that makes multiplication easier.
- Multiplying corresponding values.
- Performing modular arithmetic to keep results within the required mathematical limits.
- Managing multiple sets of polynomial data.
- Moving and storing intermediate results.

These operations all contribute to overall computation time. Our objective is to study this process, identify which parts contribute most to execution time, and investigate whether they can be made more efficient.

> **Scope boundary:** The goal is **not** to replace the CKKS encryption scheme. It is to investigate whether its underlying computational operations can be implemented more efficiently.

---

## 3. Why Are We Using OpenFHE?

OpenFHE is an open-source library that implements several FHE schemes, including CKKS. Instead of developing an encryption system from scratch, we study how an existing implementation performs its calculations. This gives three advantages:

1. We can understand how CKKS operations are implemented in an actual software library.
2. We can identify the functions responsible for computationally expensive operations.
3. We can test potential improvements against an existing implementation.

| | |
|---|---|
| **Environment** | OpenFHE **1.6.1**, built on Ubuntu Linux inside a VirtualBox environment |
| **Current stage** | Studying the existing implementation rather than modifying its core algorithms |

---

## 4. Understanding the Computation

Before attempting any optimization, we need to understand what happens when CKKS performs multiplication. A simplified view:

```text
Encrypted Data → Polynomial Operations → Modular Arithmetic → Result Processing → Encrypted Result
```

Inside polynomial multiplication, five components matter.

### 4.1 Number Theoretic Transform (NTT)

NTT converts polynomial data into another representation that makes multiplication easier and more efficient. The transformation itself takes computational time, so it is one of the components we need to understand and measure.

Our research is **not limited** to NTT. We are investigating the complete polynomial multiplication process.

### 4.2 Pointwise Multiplication

After the transformation, corresponding values are multiplied. This is one of the central operations in polynomial multiplication. Its efficiency depends on how multiplication is performed, how the results are calculated, and how the data is handled.

### 4.3 Modular Arithmetic

FHE computations use modular arithmetic, so calculations are performed within specific numerical limits. After multiplication, results may need to be reduced according to the relevant modulus. The implementation of multiplication and modular reduction may influence total execution time.

### 4.4 RNS Representation

OpenFHE also uses the **Residue Number System (RNS)**. Instead of handling a large number as a single value, RNS represents it using several smaller values associated with different moduli. This allows certain calculations to be performed more efficiently, but processing multiple sets of values also introduces additional computation and data-management requirements.

### 4.5 Memory and Data Handling

Performance is not determined only by mathematical operations. Copying data, allocating memory, accessing values, and moving intermediate results also contribute to execution time. We will investigate both the mathematical operations and the way their data is handled.

---

## 5. Our Current Understanding of OpenFHE

Tracing of polynomial multiplication inside OpenFHE has identified the following execution path:

```text
DCRT Polynomial Multiplication
        ↓
Processing individual RNS components
        ↓
Polynomial multiplication within each component
        ↓
Element-wise modular multiplication
        ↓
Modular multiplication and reduction
```

### Implementation details found so far

- OpenFHE processes different RNS components **in parallel using OpenMP**.
- The polynomial multiplication implementation uses an underlying vector operation.
- The ordinary multiplication path can involve **copying polynomial data** before the operation.
- The vector multiplication implementation performs modular multiplication on individual elements.
- The current modular multiplication is multiplication followed by **conditional modular reduction**.

These findings give specific areas to examine: unnecessary data copying, memory allocation, modular multiplication, and reduction.

> **Measure before deciding:** Identifying an operation in the source code does **not** mean it is the main bottleneck. Its actual contribution must be measured before deciding what to optimize.

---

## 6. Benchmarking: Establishing a Performance Baseline

A **baseline** is a reference point showing how long existing operations take, against which future improvements are compared.

OpenFHE's existing CKKS benchmarks:

| Benchmark | What it measures |
|---|---|
| `CKKSrns_MultNoRelin` | Ciphertext multiplication without relinearization |
| `CKKSrns_MultRelin` | Ciphertext multiplication with relinearization |
| `CKKSrns_Relin` | Relinearization as a separate operation |
| `CKKSrns_Rescale` | Modulus reduction after multiplication |

The most useful starting point is `CKKSrns_MultNoRelin`, which measures ciphertext multiplication without the relinearization step. However, it does **not** measure polynomial multiplication alone, because it still includes other operations involved in ciphertext multiplication.

We therefore plan to develop a second, more focused benchmark that directly measures multiplication between two DCRT polynomial objects.

### Two-level benchmarking approach

| Level | Goal |
|---|---|
| **1. High-level** | Measure CKKS ciphertext multiplication to see the cost of the overall operation. |
| **2. Low-level** | Measure DCRT polynomial multiplication directly to isolate the underlying operation. |

Separating the levels shows whether an improvement in a lower-level operation actually affects the performance of the larger encryption operation.

---

## 7. Proposed Optimization Areas

> **Nothing finalized:** No specific optimization has been chosen. The first objective is to investigate the existing implementation and determine which areas offer realistic opportunities for improvement.

| Area | What will be investigated |
|---|---|
| **7.1 Modular Multiplication** | How individual values are multiplied and reduced within the modulus; whether this can be more efficient without changing mathematical correctness. |
| **7.2 Memory Management** | Whether unnecessary copying, temporary storage, and memory allocation add to execution time, especially for large polynomial structures. |
| **7.3 Parallel Processing** | How OpenFHE distributes work across RNS components. Parallelism already exists, so the aim is to judge its efficiency and find opportunities at other levels, not simply add more. |
| **7.4 NTT and Polynomial Processing** | The transformation and multiplication stages studied together, keeping the overall polynomial multiplication process as the main research target. |
| **7.5 Hardware-Level Optimization** | Whether selected operations can be accelerated using FPGA hardware. The decision depends on software-level measurements. |

---

## 8. Hardware, Software and Compiler-Level Approach

Optimization is considered at three connected levels.

### Software level

Study and improve the existing OpenFHE implementation:

- More efficient modular multiplication.
- Reducing unnecessary memory operations.
- Improving polynomial data handling.
- Optimizing existing mathematical routines.

This is the **starting point**, because it allows measurable improvements before introducing additional hardware complexity.

### Hardware level

Investigate whether computationally expensive operations can run more efficiently on dedicated hardware. Modular multiplication and polynomial-processing operations are candidates for FPGA acceleration, compared against ordinary CPU execution.

### Compiler and execution level

The way computations are organized and executed can influence performance, even when the algorithm and hardware are fixed:

- Scheduling operations efficiently.
- Reducing unnecessary intermediate data movement.
- Improving data reuse.
- Mapping suitable computations to the available processing resources.

These areas become more relevant when integrating software operations with hardware acceleration.

The three levels are interconnected: improving an algorithm, changing how it executes, or moving part of it to dedicated hardware can all affect overall performance.

---

## 9. Proposed Experimental Methodology

A measurement-based approach: no change is made before its impact can be understood.

1. **Understand the existing implementation.** Study the CKKS multiplication process and trace how OpenFHE performs polynomial multiplication. Identify the relevant functions, data structures, and mathematical operations.
2. **Establish the baseline.** Run existing OpenFHE benchmarks and develop a direct polynomial multiplication benchmark. Record execution time under consistent conditions.
3. **Identify the bottleneck.** Analyze results to see which operations contribute most: modular multiplication, modular reduction, memory handling, or polynomial processing.
4. **Investigate possible improvements.** Select one specific area. Any proposed change must preserve mathematical correctness.
5. **Implement and test.** Implement the selected change separately and test it against the original implementation.
6. **Compare performance.** Run the same benchmarks before and after. Compare execution time, consistency, and correctness.
7. **Integrate and validate.** If the improvement is measurable and reliable, integrate it and evaluate its effect on the overall CKKS operation. Repeat for other promising areas.

---

## 10. FPGA-Based Testing Using PolarFire SoC

Alongside software-level investigation, we plan to explore hardware acceleration on the **PolarFire SoC FPGA development board**.

An FPGA is a programmable hardware device on which specialized circuits can be designed for particular operations. Unlike a general-purpose processor with a fixed architecture, an FPGA can be configured specifically for the operations we want to accelerate.

### Candidate operations

These are potential candidates, **not finalized hardware designs**.

- Modular multiplication
- Modular reduction
- NTT and inverse NTT
- Polynomial multiplication

### Proposed testing stages

| Stage | Description |
|---|---|
| **1. Understand the development environment** | Set up PolarFire SoC and understand how its processor and FPGA fabric communicate. |
| **2. Select a computational operation** | Choose based on software benchmarking results. Modular multiplication may be a practical start, since it is used repeatedly in polynomial multiplication. |
| **3. Develop and test the hardware implementation** | Build it and verify it produces mathematically correct results. |
| **4. Measure hardware performance** | Record execution time, resource usage, memory requirements, and data-transfer overhead. |
| **5. Compare CPU and FPGA performance** | Compare on execution time, data-transfer overhead, hardware resource usage, power consumption (where measurable), and mathematical correctness. |
| **6. Explore integration** | If the advantage is meaningful, investigate connecting it to the OpenFHE workflow in a hybrid design: the CPU handles general operations while the FPGA accelerates selected expensive ones. |

> **Important consideration:** OpenFHE currently runs as C++ software on the CPU and **cannot be transferred directly** into the FPGA fabric. The FPGA needs a separate hardware implementation of the selected operations, along with an interface for exchanging data with the processor. FPGA acceleration is therefore a **separate development stage**, not an automatic extension of the current software implementation.

---

## 11. Expected Evaluation

| Parameter | Purpose |
|---|---|
| **Execution Time** | Measures how long an operation takes |
| **Latency** | Measures the time required to complete a computation |
| **Throughput** | Measures how much computation can be completed over a period of time |
| **Memory Usage** | Helps identify the cost of storing and moving intermediate data |
| **Hardware Resource Usage** | Measures FPGA resource requirements |
| **Power Consumption** | Helps evaluate the energy cost of hardware execution |
| **Correctness** | Ensures that optimized operations produce valid results |

The original implementation is compared against any modified implementation under consistent test conditions.

> **Success criterion:** An optimization is successful only if it provides a **measurable improvement without compromising correctness**.

---

## 12. Current Progress and Immediate Next Steps

The project is in the investigation and baseline-establishment stage.

### Completed

- [x] OpenFHE 1.6.1 cloned and built successfully on Ubuntu.
- [x] Existing CKKS benchmark functions located.
- [x] Source-code path for DCRT polynomial multiplication traced.
- [x] RNS components and OpenMP parallel processing identified.
- [x] Underlying vector-level modular multiplication implementation examined.
- [x] Distinction between ciphertext multiplication, relinearization, and polynomial multiplication established.

The latest benchmark source investigation identified:

- `CKKSrns_MultNoRelin` for ciphertext multiplication without relinearization.
- `CKKSrns_MultRelin` for multiplication with relinearization.
- `CKKSrns_Relin` for measuring relinearization separately.

These existing benchmarks will help establish the higher-level performance baseline.

### Immediate next steps

- [ ] Examine the CKKS benchmark configuration and understand the parameters used during testing.
- [ ] Establish a stable and repeatable baseline.
- [ ] Create a focused benchmark for direct DCRT polynomial multiplication.
- [ ] Verify the actual modulus sizes and numerical representation used by the active implementation.
- [ ] Profile the multiplication process to identify the most significant sources of latency.
- [ ] Select one realistic optimization target and test it independently.

> **No optimization has been finalized or claimed at this stage.**

---

## 13. Project Scope and Long-Term Direction

The current focus is on understanding and improving the computational performance of polynomial multiplication within the CKKS implementation provided by OpenFHE.

```text
Software analysis → Software optimization → FPGA acceleration → CPU–FPGA integration
```

- The research initially concentrates on software-level analysis and optimization.
- Based on the findings, hardware acceleration will be investigated using the PolarFire SoC FPGA development board.
- Compiler-level improvements and CPU–FPGA integration may also be explored as the project develops.

The overall direction is to investigate how software algorithms, hardware resources, and execution strategies can work together to reduce the computational overhead of Fully Homomorphic Encryption.

The project will remain **experimental and measurement-driven**. The exact optimization technique will be selected only after the existing implementation has been properly analyzed.

### Central research question

> Can the computational latency of CKKS polynomial multiplication be reduced through improvements in its underlying mathematical operations, software implementation, and hardware-level execution?
