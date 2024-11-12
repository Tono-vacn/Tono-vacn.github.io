---
layout: post
title: Parallel Programming Memo
subtitle: Try to do it in parallel
# cover-img: /assets/img/path.jpg
thumbnail-img: /assets/img/parallel-cover.png
# share-img: /assets/img/path.jpg
tags: [
    Parallel Programming,
    OpenMP,
    Code Profiling,
]
author: Yuchen Jiang
readtime: true
---

## Creating a Parallel Program

- Task Creation:  identifying parallel tasks, variable scopes, synchronization
  - Finding parallel tasks:
    - Code analysis
    - Algorithm analysis
  - Viriable partitioning:
    - Shared variables
    - Private variables
    - Reduction variables
  - Syncronization: 
- Task Mapping: grouping tasks, mapping to processors/memory
  - Static & Dynamic Mapping
  - Block & Cyclic Mapping
  - Dimension Mapping: column-wise or row-wise
  - Communication and Data locality considerations 

### Code Analysis

  - Focus mainly on loop dependence analysis
    - True, Anti, Output dependence
    - Loop-carried vs. Loop independent
    - ITG, LDG
  - Example:
    ![example3](/assets/img/parallel-example3.png) 
    ![example4](/assets/img/parallel-example4.png)
    ![example5](/assets/img/parallel-example5.png)

  - DOACROSS Parallelization
    - Dependecy
      ```c
      for (i=1; i<=N; i++) {
        S: a[i] = a[i-1] + b[i] * c[i];
      }
      ```
      S[i] =>T S[i+1]
      ![DOACROSS](/assets/img/parallel-DOACROSS.png)
  - DOPIPE Parallelization
    - Dependecy
      ```c
      for (i=2; i<=N; i++) {
        S1: a[i] = a[i-1] + b[i];
        S2: c[i] = c[i] + a[i];
      }
      ```
      S1[i] =>T S2[i]
      
      S1[i] =>T S1[i+1]
      ![DOPIPE](/assets/img/parallel-DOPIPE.png)


### Algorithm Analysis

  code analysis misses parallelization opportunities available at the algorithm level

### Determining Viriable Scope

- Basic Category:
  - Read-Only: variable is only read by all tasks
  - R/W non-conflicting: variable is read, written, or both by only one task
  - R/W Conflicting: variable written by one task may be read by another

  - example:
    - ```c
      for (i=1; i<=n; i++)
        for (j=1; j<=n; j++) {
          S1: a[i][j] = b[i][j] + c[i][j];
          S2: b[i][j] = a[i][j-1] * d[i][j];
          S3: c[n][j] = d[i-1][j];
        }
      ```
      Define a parallel task as each “for i” loop iteration
      
      Here `i` is a R/W conflicting variable, since each task will take control of several `i` values, which means they will write to the same variable.


    - ```c
      for (i=1; i<=n; i++)
        for (j=1; j<=n; j++) {
          S1: a[i][j] = b[i][j] + c[i][j];
          S2: b[i][j] = a[i-1][j] * d[i][j];
          S3: e[i][j] = a[i][j];
        }
      ```
      Parallel task = each “for j” loop iteration

      Here `j` is a R/W conflicting variable, same reason as above.

<!-- - ![variable](/assets/img/parallel-variable.png) -->
- Variable Privatization
  - Conflicting variable that, in program order, is always **defined (=written) by a task before use (=read) by the same task**
  - Conflicting variable whose values for different parallel tasks are **known ahead of time** (hence, private copies can be initialized to the known values)
  - making private copies of a shared variable
  - R/W Conflicting => R/W Non-conflicting
  - Reduction Variables and Operation
    - Reduces elements of some vector/array down to one element: sum, multiplication, logical
    - Reduction variable is updated by each task, and the **order of update is not important**
    - commutative and associative
  - How we do it:
    - Should be declared shared:
      -   Read-only variables
      - R/W Non-conflicting variables
    - Should be declared reduction:
      - Reduction variables
    - Other R/W conflicting variables:
      - Privatization possible? If so, privatize them
      - Otherwise, declare as shared, but protect with synchronization

### Task Mapping
- Static and Dynamic Mapping
  -  Static： tasks are assigned to threads ahead of time
     - lower runtime overhead  
  -  Dynamic: tasks are assigned to threads at runtime
     -  better load balancing
     -  need shared task queue
- Block and Cyclic Mapping
  - Cyclic: Threads are assigned tasks in a striped fashion
    - 1st thread: 1st, 1+n, 1+2n, …
    - 2nd thread: 2nd, 2+n, 2+2n, … 
  - Block: Threads are assigned contiguous chunks of tasks
    - 1st thread: 1st, 2nd, …, n
    - 2nd thread: n+1, n+2, …, 2n
  - Consider both data locality and communication overhead
    - Want to exchange minimal data between threads
    - Also want to maximize locality
  
## OpenMP Programming

- OpenMP: a set of compiler directives, library routines, and environment variables that can be used to specify high-level parallelism in Fortran and C/C++ programs

### OpenMP Usage

```c
#pragma omp directive-name [clause[ [,] clause]...] new-line
//structured-block
```

- directive-name: the name of the directive
  - parallel: create a team of threads
  - for: distribute loop iterations among threads
  - sections: distribute sections among threads
  - section: define a section of code to be executed by a single thread
  - single: execute a block of code in a single thread
  - target: offload code to a target device 
- clause: a directive-specific option
  - private: private variables
  - shared: shared variables
  - reduction: reduction variables`reduction(reduction-identifier: list)`
- Type of variables
  - shared: shared among all threads
  - private: each thread has its own copy
  - firstprivate: private copy initialized with the value of the original variable
  - lastprivate: private copy value of last iteration is copied back to the original variable
  - reduction: variable that is the target of a reduction operation performed by the loop, e.g., sum 
    - example:
      ```c++
      #include <omp.h>
      {
        int sum = 0;
        #pragma omp parallel for default(shared) private(i) reduction(+:sum)
        for(i=0; i<n; i++)
          sum = sum + A[i];
      }
      ``` 
      ```c++
      #include <omp.h>
      {
        int sum = 0;
        int local_sum[num_threads];
        #pragma omp parallel for default(shared) private(i)
        for(i=0; i<n; i++)
          local_sum[thread_id] = local_sum[thread_id] + A[i];
        for(i=0; i<num_threads; i++) 
          sum += local_sum[i];
      }
      ```
      so basically, these two code snippets are doing the same thing, and the `reduction` clause allocate a copy of ‘sum’ per thread and combine per-thread copies into original ‘sum’ at the end
- Barriers
  - implicit after each parallel section
  - use `nowait` clause to avoid barrier
      ```c++
      #include <omp.h>
      //…
      {
        //…
        #pragma omp for default(shared) private(i) nowait
        for(i=0; i<n; i++)
          A[i]= A[i]*A[i]- 3.0;
      }
      ```     

### OpenMP API functions
- `omp_get_num_threads()`: return the number of threads in the current team
- `omp_set_num_threads()`: set the number of threads to be used in the next parallel region
- `omp_init_lock()`: initialize a simple lock
- `omp_set_lock()`: acquire a simple lock
- `omp_unset_lock()`: release a simple lock


## Parallel Programming Issuses

- Correctness
  -  Result preservation
  -  Synchronization points and types
  -  Variable partitioning
- Performance
  -  Task granularity
  -  Syncronization granularity
  -  Lack of utilization of reduction variables

### Result Preservation
- example: rounding problem of floating-point numbers, the order of summation can affect the result, which is a problem in parallel programming

### Synchronization Points and Types
- Race Condition

### Variable Partitioning
- example
  ```c++
    #pragma omp parallel for shared(b,temp) private(a,c)
    for (i=0; i<N; i++) {
      temp = b[i] + c[i];
      a[i] = a[i] * temp;
    }
  ```
  - c should be shared but declared private
    - storage overhead
    - outcome is non-deterministic, depends on whether the private c is initialized to the global c
  - a should be shared but declared private
    - storage overhead
    - outcome non-deterministic, depends on whether the private a is merged into the global a
  - temp should be declared private but declared shared
    - wrong output due to threads overwriting temp

### Parallel Task Granularity