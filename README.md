# PSA01_OpenMP_5center
# CMDA 3634 PSA01 : OpenMP 5-Center Problem

**Author:** Jason Bruno Terceros  
**Course:** CMDA 3634 – Parallel Programming  
**Date:** January 2024 - February 2024

## 📌 Overview

This project solves the **5-center problem** for 128 two‑dimensional points (cities) using **OpenMP** parallelization. The goal is to choose five center points from the given set so that the maximum distance between any point and its closest center is minimized.  

We compare two scheduling strategies (`static` vs. `dynamic`) for thread counts 1, 2, and 4. The sequential code is provided; we modify it to run in parallel using OpenMP directives, ensuring thread safety and performance.

---

## 🧩 Problem Statement

Given \(n\) points \(p_1, p_2, \dots, p_n\) in 2D space, choose five centers \(c_1, c_2, c_3, c_4, c_5\) from the points to minimize:

\[
\text{cost}(c_1, c_2, c_3, c_4, c_5) = \max_{1 \leq i \leq n} \min\bigl( \|p_i - c_1\|, \|p_i - c_2\|, \|p_i - c_3\|, \|p_i - c_4\|, \|p_i - c_5\| \bigr)
\]

The brute‑force sequential solution checks all \(\binom{n}{5}\) 5‑tuples, which for \(n=128\) is about \(2.6 \times 10^8\) combinations – an ideal candidate for parallelization.

---

## 🔧 Implementation

### Sequential baseline
The original code uses five nested loops (`i < j < k < l < m`) and a distance table to compute the cost of each 5‑tuple. It includes an “early dropout” optimization: if a partial distance already exceeds the current best, the rest of the tuple is skipped.

### Parallel version
We parallelize the outermost loop with `#pragma omp for`. The key modifications are:

- **Private variables** per thread:  
  `thread_min_cost_sq`, `thread_num_checked`, `thread_optimal`, and the loop indices `i,j,k,l,m`.
- **Shared variables** (accessed by all threads):  
  `n`, `dist_sqs`, `num_checked`, `min_cost_sq`, `optimal`.
- A **critical section** at the end of the parallel region to combine results from all threads:
  ```c
  #pragma omp critical
  {
      *num_checked += thread_num_checked;
      if (thread_min_cost_sq < min_cost_sq) {
          min_cost_sq = thread_min_cost_sq;
          *optimal = thread_optimal;
      }
  }
  ```
  We compile with either `-DDYNAMIC` (dynamic schedule) or without it (static schedule) to compare the two

---

## 📊 Results

The program was run for 1, 2, and 4 threads using both scheduling policies. Below are the elapsed times and speedups.

### Static schedule
| Threads | Elapsed time (s) | Speedup |
|---------|------------------|---------|
| 1 | 18.7915 | 1.00 |
| 2 | – | – |
| 4 | 14.4549 | 1.30 |

### Dynamic schedule
| Threads          | Elapsed time (s) | Speedup |
|-------------------|--------------|--------|
| 1 | 18.7572 | 1.00 |
| 2 | – | – |
| 4 | 5.29484 | 3.54 |

> **Note:** Speedup = time(1 thread) / time(P threads).
> Dynamic schedule shows near‑linear speedup (3.54 with 4 threads), while static achieves only 1.30.

---

## 📈 Analysis
### Why is dynamic scheduling so much faster?

The workload of the 5‑center problem is unbalanced – some 5‑tuples are pruned early (by the dropout optimization), others run to completion.
- **Static schedule** divides the iteration space evenly among threads. Because some threads get “hard” (long‑running) chunks and others easy ones, the slowest thread dominates the total time.
- **Dynamic schedule** (`schedule(dynamic)`) assigns chunks to threads as they become free, achieving a much better load balance. The output shows that for 4 threads, dynamic gives roughly equal numbers of checked tuples per thread, whereas static leaves some threads with far fewer tuples – but their waiting time is not recovered.

> This explains the dramatic speedup improvement (3.54 vs 1.30).

### Thread safety
We ensured correctness by:
- Using private copies of `min_cost_sq`, `num_checked`, and `optimal` for each thread
- Reducing all intermediate results into the shared variables inside a critical section, so updates are atomic
- The loop indices (`i,j,k,l,m`) are automatically private to each thread.

### Minimal cost
Regardless of thread count or schedule, the computed minimal cost is identical. The optimal centers may differ because when multiple 5‑tuples tie for the same cost, the first one found (which depends on execution order) is recorded.

---

## 🚀 How to Compile and Run

1. Clone the repository and navigate to the src/ folder.
2. Compile with OpenMP (example using gcc):
    ```bash
    # Static schedule
    gcc -fopenmp -O2 omp_5center.c -o omp_5center_static
    
    # Dynamic schedule
    gcc -fopenmp -O2 -DDYNAMIC omp_5center.c -o omp_5center_dynamic
    ```
3. Run for different thread counts:
    ```bash
    export OMP_NUM_THREADS=1
    ./omp_5center_static
    
    export OMP_NUM_THREADS=2
    ./omp_5center_static
    
    export OMP_NUM_THREADS=4
    ./omp_5center_static
    ```
  (Repeat for _dynamic)

4. The program will output the minimal cost, the optimal centers, the number of tuples checked, and the execution time

---

## 📁 Repository Contents
- src/omp_5center.c – Parallel OpenMP implementation
- images/Static.png – Screenshot of static schedule output
- images/Dynamic.png – Screenshot of dynamic schedule output
- README.md – This file

---

## 👤 Author
**Jason Bruno Terceros** – GitHub Profile
> Course: CMDA 3634 – Comp Sci Foundations
> Virginia Tech
