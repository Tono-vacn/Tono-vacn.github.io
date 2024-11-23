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

## Table of Contents
- [Table of Contents](#table-of-contents)
- [Creating a Parallel Program](#creating-a-parallel-program)
  - [Code Analysis](#code-analysis)
  - [Algorithm Analysis](#algorithm-analysis)
  - [Determining Viriable Scope](#determining-viriable-scope)
  - [Task Mapping](#task-mapping)
- [OpenMP Programming](#openmp-programming)
  - [OpenMP Usage](#openmp-usage)
  - [OpenMP API functions](#openmp-api-functions)
  - [OpenMP SCHEUDLE directive](#openmp-scheudle-directive)
  - [OpenMP Memory Model](#openmp-memory-model)
    - [Ensure Consistent View of Memory](#ensure-consistent-view-of-memory)
  - [OpenMP Task Directive](#openmp-task-directive)
- [Parallel Programming Issuses](#parallel-programming-issuses)
  - [Result Preservation](#result-preservation)
  - [Synchronization Points and Types](#synchronization-points-and-types)
  - [Variable Partitioning](#variable-partitioning)
  - [Parallel Task Granularity](#parallel-task-granularity)
  - [Syncronization Granularity](#syncronization-granularity)
- [Inherent vs. Artifactual Communication](#inherent-vs-artifactual-communication)
  - [Communication-to-Computation Ratio](#communication-to-computation-ratio)
  - [Memory Hierachy Issue](#memory-hierachy-issue)
    - [Exploiting Spatial Locality at Page Level](#exploiting-spatial-locality-at-page-level)
- [Syncronization](#syncronization)
- [Pthread](#pthread)
- [Parallel Programming for Linked Data Structures](#parallel-programming-for-linked-data-structures)
- [Memory Consistency](#memory-consistency)
  - [Programmers' Intuition](#programmers-intuition)
  - [Building SC systems](#building-sc-systems)
  - [Relaxed Consistency Models](#relaxed-consistency-models)
    - [Implementation of Memory Fence](#implementation-of-memory-fence)
    - [Processor Consistency (PC)](#processor-consistency-pc)
- [Lock Free Data Structures](#lock-free-data-structures)
  - [C++ Memory Model Support](#c-memory-model-support)
  - [Lock-Free Data Structures](#lock-free-data-structures-1)
    - [Lock-Free LinkedList::addFront](#lock-free-linkedlistaddfront)
    - [Lock-Free LinkedList::addSorted](#lock-free-linkedlistaddsorted)
    - [Lock-Free LinkedList::removeFront](#lock-free-linkedlistremovefront)
- [Programming for Clusters](#programming-for-clusters)
  - [MPI (Message Passing Interface)](#mpi-message-passing-interface)
  - [MapReduce](#mapreduce)
    - [OverView](#overview)


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

Forms a team of threads and starts parallel execution, Thread that enters the region becomes the **master**

![OpenMPVisual](/assets/img/openMP-visual.png)

After parallel block, the thread team **sleeps** until needed

  ```c++
    #include <omp.h>
    {
    omp_set_num_threads(4);
    #pragma omp parallel
      {
        #pragma omp for private(i), shared(N, A, B, C)
        for (int i=0; i<N; i++) {
          A[i] = B[i] + C[i];
        }
      }
    }
  ```

  ```c++
    #include <omp.h>
    {
      omp_set_num_threads(4);
      #pragma omp parallel for private(i), shared(N, A, B, C)
      for (int i=0; i<N; i++) {
        A[i] = B[i] + C[i];
      }
    }
  ```

The above two code snippets are doing the same thing

- directive-name: the name of the directive
  - parallel: create a team of threads
  - for: distribute loop iterations among threads
  - sections: distribute sections among threads
  - section: define a section of code to be executed by a single thread
  - single: execute a block of code in a single thread
    - only one thread executes the block, no guarantee which thread
    - `#pragma omp single`
  - master: execute a block of code in the master thread
    - only the master thread executes the block
    - `#pragma omp master`
  - target: offload code to a target device
  - critical: ensure that only one thread executes a block of code at a time
    - `#pragma omp critical (name)`
  - atomic: ensure that a block of code is executed atomically
    - `#pragma omp atomic [read | write | update | capture]`
      - read expression: `v = x;`
      - write expression: `x = expr;`
      - update expression: `x++;`
      - capture expression: `V = x++;`
    -  Ensure a specific storage location is updated atomically
- clause: a directive-specific option
  - private: private variables
  - shared: shared variables
  - reduction: reduction variables`reduction(reduction-identifier: list)`
  - collapse: collapse nested loops into one loop
    - at least 2
  - ordered: enforce order inside a loop
    - ordered directives are executed same way as the sequential loop would execute them (one at a time)
    - Each loop iteration may execute at most one ordered directive
    - example
      ```c++
        void work(int i) {
          printf(“i = %d\n”, i);
        }
        //  ...
        int i;
        #pragma omp parallel for ordered
        for (i=0; i < 20; i++) {
          #pragma omp ordered
          work(i);
          //Each iteration of the loop has 2 ordered clauses!
          #pragma omp ordered
          work(i+100);
        }
      ``` 

      ```c++
        void work(int i) {
          printf(“i = %d\n”, i);
        }
        //  ...
        int i;
        #pragma omp parallel for ordered
        for (i=0; i < 20; i++) {
          if (i <= 10) {
            #pragma omp ordered
            work(i);
          }
          if (i > 10) {
          //two ordered clauses are mutually exclusive
            #pragma omp ordered
            work(i+100);
          }
        }
        //if we change (i > 10) to (i > 9), it becomes invalid
      ```
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

### OpenMP SCHEUDLE directive

- `Static`
- `Dynamic`: tasks and central task queue
- `Guided`: except that the task sizes are not uniform, early tasks are exponentially larger
- `Runtime`: Check the environment variable `OMP_SCHEDULE` at run time to determine what scheduling to use

- `SCHEDULE(Static | Dynamic | Guided | Runtime, chunksize)`
  ```c++
    sum = 0;
    pragma omp parallel for reduction(+:sum) schedule(static, n/p) 
    {
    for (i=0; i<n; i++)
      for (j=0; j<i; j++)
        sum = sum + a[i][j];
    print sum;
    }
  ```

  ![chunksize](/assets/img/parallel-chunksize.png)
  ![chunksize2](/assets/img/parallel-chunksize2.png)
  ![chunksize3](/assets/img/parallel-chunksize3.png)

### OpenMP Memory Model

- relaxed-consistency
  - Each thread can have its own temporary view of memory
  - thread’s temporary view of memory is not required to be consistent with memory
  - Update to shared memory from one thread may not be seen by the other thread “immediately”
- shared memory
  - All threads share a single store called memory
  - May not actually represent RAM

#### Ensure Consistent View of Memory

- `flush`: ensure that all threads have a consistent view of memory
  - `#pragma omp flush(list)`
  - `list` is a comma-separated list of variables
  - example
    ```c++
      #pragma omp parallel
      {
        //…
        #pragma omp flush(A)
        //…
      }
    ```
  - orders: 
    - Those before the flush complete before the flush executes
    - Those after the flush complete after the flush executes
    - Flushes with overlapping flush-sets cannot be reordered
      ```c++
        void thread1() {
            shared_var = 42;   
            #pragma omp flush(shared_var) 
        }

        void thread2() {
            int local_var;
            #pragma omp flush(shared_var) 
            local_var = shared_var;
            printf("Thread 2 reads shared_var: %d\n", local_var);
        }
      ```

### OpenMP Task Directive

`#pragma omp task [clause[ [,] clause]...]` 
- Generates a task for a thread in the team to run
- When a thread enters the region it may
  - Immediately execute the task, or
  - Defer its execution (another thread may be assigned the task)
- clauses:
  - if, final, untied, default, mergeable, private, firstprivate, shared
  - `if`: When expression is false, generates an undeferred task
  - `final`: When expression is true, generates a final task
    - All tasks within a final task are included
    - Included tasks are undeferred and also execute immediately in the same thread
  - `untied`: Task is not tied to the thread that created it
  - `mergeable`: Task can be merged with the current task
- Three Logical Task Concepts:
  - Deferred Task: generates a new task, Put the task on the task queue for any thread to execute
  - Undeferred Task: suspends its current task, executes newly encountered task, may resume its previously suspended task
  - Included Task: simply merges the task into its current task

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

![granularity](/assets/img/parallel-granularity.png)

- Easy way to increase parallel thread granularity: parallelize the outer loop rather than the inner loop
- But must watch out for load balance 

### Syncronization Granularity

- Linked list example:
  - coarse-grained: lock the entire list
  - fine-grained: lock each node

## Inherent vs. Artifactual Communication

Overhead and Scalability factors in parallel programming
- Important metric:
  - communication to computation ratio
  - Use this to infer the scalability of a parallel program
- inherent communication
  - Caused by task-to-process mapping
- actual communication
  - Caused by process-to-processor mapping

### Communication-to-Computation Ratio

![ocean-example](/assets/img/parallel-ocean.png)

Here in the example we assume three different ways to map the nodes into processors. Each side of the square indicate N elements.

- Block-wise partitioning
  
    $$
      CCR = \frac{\text{Comm}}{\text{Comp}} = \frac{\frac{4n}{\sqrt{p}}}{\frac{n^2}{p}} = \frac{4 \sqrt{p}}{n}
    $$

- Row-wise partitioning
  
    $$
      CCR = \frac{\text{Comm}}{\text{Comp}} = \frac{\frac{2n}{\sqrt{p}}}{\frac{n^2}{p}} = \frac{2p}{n}
    $$

### Memory Hierachy Issue

key to performance: spatial and temporal locality
at each hierarchy level
- Registers, caches, local memory, remote memory (topology)
- Lower level: higher size, higher communication cost

#### Exploiting Spatial Locality at Page Level

- Page distribution policy in NUMA:
  - Single node: all pages in a single node, lack of spatial locality, contention
  - Round-robin: pages are allocated in consecutive nodes, lack of spatial locality, lower contention
  - First touch: pages are allocated in the node where the first write occurs, better spatial locality, lower contention
  - Page Migration: Monitor and migrate pages at runtime

## Syncronization

- Race Condition
  - Result of computation by concurrent threads depends on the precise timing of the execution of an instruction sequence by one thread relative to another
  - Non-deterministic result
- **Global Event Synchronization**
  - `BARRIER (name, nprocs)`:Thread will wait at barrier call until nprocs threads arrive
    - Built using lower level primitives
    - Heavyweight operation
  - Synchronize across a subset of all parallel processes
    - Done using flags or barriers
    - Basic concept of producers and consumers
      - Single producer, multiple consumer
      - Multiple producer, single consumer
      - Multiple producer, multiple consumer
- **Point-to-point Event Synchronization**
  - thread notifies another thread so it can proceed
    - Typical in producer-consumer behavior
    - Concurrent programming on uniprocessors: semaphores
    - semaphores or monitors or variable flags

## Pthread

- POSIX pthreads: a standard for thread creation and management
- Basic Usage: 
  - `#include <pthread.h>`
  - `gcc –o p_test p_test.c –lpthread`
- Create a pthread:
  - `int pthread_create(pthread_t *thread, pthread_attr_t *attr, void
*(*start_routine)(void *), void *arg);`
  - `pthread_t` is the thread identifier
  - `pthread_attr_t` is the thread attributes
  - `start_routine` is the function to be executed by the thread
  - `arg` is the argument to the function
  - example 
    ```c++
      pthread_t *thrd = (pthread_t*)malloc(sizeof(pthread_t));
      pthread_create(thrd, NULL, &do_work_fcn, NULL);
      //Or
      pthread_t thrd;
      pthread_create(&thrd, NULL, &do_work_fcn, NULL);
    ```
- Pthread Destruction
  - `pthread_join(pthread_t thread, void **retval);`
    - Blocks the calling thread until the specified thread terminates
    - `retval` is the return value of the thread
  - `pthread_exit(void *value_ptr);`
    - Terminates the calling thread
    - `value_ptr` ：Provides value_ptr to any pending pthread_join() call
- Pthread Mutex
  - `pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;`
  - `int pthread_mutex_init(pthread_mutex_t *mutex, const pthread_mutexattr_t *mutex_attr);`
  - `pthread_mutex_lock(&mutex);`
  - `pthread_mutex_trylock(&mutex);`
  - `pthread_mutex_unlock(&mutex);`
  - `pthread_mutex_destroy(&mutex);`  
- Pthread Condition Variable
  - `pthread_cond_t cond = PTHREAD_COND_INITIALIZER;`
  - `int pthread_cond_init(pthread_cond_t *cond, const pthread_condattr_t *cond_attr);`
  - `pthread_cond_wait(&cond, &mutex);`
  - `pthread_cond_signal(&cond);`
  - `pthread_cond_broadcast(&cond);`
  - `pthread_cond_destroy(&cond);`
  - Always used in conjunction with a mutex lock
    ```c++
      void *thr_func1(void *arg) {
      /* thread code blocks here until MAX_COUNT is reached */
        pthread_mutex_lock(&count_lock);
        while (count < MAX_COUNT)
          pthread_cond_wait(&count_cond, &count_lock);
        pthread_mutex_unlock(&count_lock);
        /* proceed with thread execution */
        pthread_exit(NULL);
      }
      /* some other thread code that signals a waiting thread that
      MAX_COUNT has been reached */
      void *thr_func2(void *arg) {
        pthread_mutex_lock(&count_lock);
        /* some code here that does interesting stuff and modifies count */
        if (count == MAX_COUNT) {
          pthread_mutex_unlock(&count_lock);
          pthread_cond_signal(&count_cond);
        } else {
          pthread_mutex_unlock(&count_lock);
        }
        pthread_exit(NULL);
      }
    ```
- Pthread Barrier
  - ` pthread_barrier_t barrier = PTHREAD_BARRIER_INITIALIZER(count);`
  - `int pthread_barrier_init(pthread_barrier_t *barrier, const pthread_barrierattr_t *attr, unsigned count);`
  - `int pthread_barrier_wait(pthread_barrier_t *barrier);`

## Parallel Programming for Linked Data Structures

- Split traversal step from processing step(s)
- Master thread traverses the list
- Creates a child thread to process each node as required

- Parallel Strategy
  - Global lock approach
    - Each operation logically has two steps
      - Traversal
      - Modification
    - perform the traversal in parallel, but list modification in a critical section
      ```c++
        IntListNode_Insert(node *p) {
          /* perform traversal */
          acq_write_lock();
          /* then check validity: 
            (1) nodes still there,
            (2) link still valid */
          /* if not valid, repeat traversal */
          /* if valid, modify list */
          if (prev->next != p || prev->deleted || p->deleted)
            /* repeat list traversal step */
          else
            /* insert node */
          //…
          rel_write_lock();
        }
      ```
  - Fine-grained lock approach
    - Associate each node with a lock (read, write)
    - (Read and write) operations execute in parallel except when they conflict on some nodes
    - Deadlocks can be avoided by imposing a global lock acquisition order
      - Careful ordering of lock acquisition must be enforced: always acquire locks for nodes from left to right in list
        ```c++
          void Insert(pIntList pList, int x) {
            int succeed;
            // traversal code to find insertion point
            succeed = 0;
            do {
              acq_write_lock(prev);
              acq_read_lock(p);
              if (prev->next != p || prev->deleted || p->deleted) {
                rel_write_lock(prev);
                rel_read_lock(p);
              //Repeat traversal; return if not found
              } else
              succeed = 1;
            } while (!succeed)

            newNode->next = p;
            if (prev != NULL)
              prev->next = newNode;
            else
              pList->head = newNode;
            rel_write_lock(prev); rel_read_lock(p);
          }
        ```

## Memory Consistency

- Cache Coherence
  - deals with ordering of writes to a single memory location
  - only needed for systems with caches
- Memory consistency
  - deals with ordering of reads/writes to all memory locations
  - concerns systems with or without caches

### Programmers' Intuition

- Program order expectation
  - programmers expect that the order in which memory accesses are executed in a thread follows the order in which they occur in the source code
- Atomicity expectation
  - An expectation that a read/write happens instantaneously/instantly with respect to all processors

- Memory accesses coming out of a processor should be performed in program order, and each of them should be performed atomically

- Sequential Consistency
  - programmers' expectations have been found to fit closely to Sequential Consistency (SC)
  - ![SC](/assets/img/memory-SC.png)
  

### Building SC systems

- Program order
  - compiler does not reorder memory accesses
  - Declare critical variables as volatile
- Atomicity
  - Execute one memory access one at a time, in program order
- Performance: Limited by Strict Ordering Requirements
    
  ![SC-implications](/assets/img/memory-SC-implications.png)

### Relaxed Consistency Models

Allows “some” memory accesses to be reordered or overlapped
- Safety Nets
  - Fence instruction ensures ordering between preceding load/store and following load/store
- Relaxed consistency models
  - Processor Consistency (PC)
  - Weak ordering (WO)
  - Acquire / Release Consistency (RC)
  -  Lazy Release Consistency (LRC)

#### Implementation of Memory Fence

- Fence ensures that mem ops that are younger are not issued until the older mem ops have globally performed
  - Wait until all older writes have been posted on the bus
  - Wait until all older reads have completed
  - Flush the pipeline to avoid issuing younger mem ops early

#### Processor Consistency (PC)

  ![PC](/assets/img/PC.png)


## Lock Free Data Structures

### C++ Memory Model Support
In C++ 11:
- `std::atomic<T>`: atomic operations on type T
  - `load()`, `store()`
    - Default mode for loads/stores to atomics is sequential consistency
      - Likely expensive to enforce
      - compiler may just insert fences (memory barriers)
    - `std::memory_order_relaxed`
      - Atomicity is provided, but any reordering is possible
      - Much Faster
    - `std::memory_order_acquire`
      - Atomicity is provided, and also ordering
      - less overhead in a general program

### Lock-Free Data Structures

```c++
  class LinkedList {
    class Node{
    public:
      const int data;
      std::atomic<Node *> next;
      Node(int d): data(d), next(nullptr) { }
      Node(int d, Node * n): data(d), next(n) { }
      ~Node() { }
    };
    std::atomic<Node*> head;
    //...
  };
```

#### Lock-Free LinkedList::addFront

![Lock-free-list](/assets/img/lock-free-linked.png)

- `std::memory_order_acq_rel` for `head.compare_exchange_weak()`
  - Acquire and Release Semantics

    - Acquire semantics
      - Ensures that all memory operations before the acquire operation are completed before the acquire operation
    - Release semantics
      - Ensures that all memory operations after the release operation are completed after the release operation

    ![Acquire-Release](/assets/img/Acquire-release.png)

  - `std::memory_order_acq_rel`
    - Both acquire and release semantics
      - Acquire semantics for the load: can see all memory writes before the load
      - Release semantics for the store: momery operations of current thread are visible to other threads after the store
    - Why `std::memory_order_acq_rel`?
      - Load: will read head pointer, need to ensure head is the latest value (Acquire)
      - Store: If head is updated successfully, need to ensure that all memory operations of current thread are visible to other threads (Release)
- `std::memory_order_relaxed` for `temp->next.store`
  - temp is private to the thread
  - next step is CAS with `std::memory_order_acq_rel`, will ensure that the store is visible to other threads
  - So we can use `std::memory_order_relaxed` here
- `std::memory_order_relaxed` for `head.load()`
  - CAS will perform acquire, and fail if head is changed
  - so simply load head with `std::memory_order_relaxed`

#### Lock-Free LinkedList::addSorted

![Lock-free-list2](/assets/img/lock-free-linked-addSorted.png)

- Why not use `std::memory_order_seq_cst` for all operations?
  - right for correctness, but too expensive

#### Lock-Free LinkedList::removeFront

![Lock-free-list3](/assets/img/lock-free-linked-removeFront.png)

![Lock-free-list4](/assets/img/lock-free-linked-removeFront_Exp.png)

- Difficult to clean up memory
  - Delete later as no other thread still references it
  - Possibility for recycle: only if no other thread is referencing it

- Solutions:
  - ![Lock-free-list5](/assets/img/lock-free-memory-freeing.png)
  - GC
    - Stop the world
    - Consider root sets from all threads

## Programming for Clusters

Distributed vs. Shared Memory
- Shared memory
  - Communication: implicit
  - Synchronization: explicit
- Distributed (clusters)
  - Communication: explicit
  - Synchronization: implicit

### MPI (Message Passing Interface)

A language-independent communication protocol for parallel-computers
- Provides explicit message passing between nodes
- ![MPI](/assets/img/MPI.png)
- Uses SPMD computation model
  - Single program, multiple data
  - Multiple instances of a single program
  - All work on different data at the same time
- MPI facilitates data communication between processes
- Program could run on one machine or cluster of machines
  - one machine: using MPI for IPC
  - cluster: using MPI for network communication

- MPI Functions
  ```c++
    // Initialize MPI
    int MPI_Init(int *argc, char **argv)
    // Determine number of processes within a communicator
    int MPI_Comm_size(MPI_Comm comm, int *size)
    // Determine processor rank within a communicator
    int MPI_Comm_rank(MPI_Comm comm, int *rank)
    // Exit MPI (must be called last by all processors)
    int MPI_Finalize()
    // Send a message
    int MPI_Send(void *buf, int count, MPI_Datatype datatype, int dest,
    int tag, MPI_Comm comm)
    // Receive a message
    int MPI_Recv(void *buf, int count, MPI_Datatype datatype，int source,
    int tag, MPI_Comm comm, MPI_Status *status)
  ```
  - `MPI_Comm`: communicator, `MPI_COMM_WORLD` for global channel
  - `MPI_Datatype`: enum for data type, like `MPI_INT`
  - `dest/source`: The “rank” of the process to send a message to / receive from
  - Both MPI_Send and MPI_Recv are blocking calls
  - tag allows for message organization, like filtering

- Simple Example
  -   ```c++
        #include <stdio.h>
        #include <mpi.h>
        int main (int argc, char *argv[])
        {
          int rank, size;

          MPI_Init(&argc, &argv);
          MPI_Comm_rank(MPI_COMM_WORLD, &rank); //get current process id
          MPI_Comm_size(MPI_COMM_WORLD, &size); //get number of processes
          printf(“Hello World from process %d of %d\n”, rank, size);
          MPI_Finalize();
          return 0;
        }
      ```
  - Controller-worker paradigm
    - Controller (rank 0) process: Creates strings, Sends them to worker processes
    - Worker processes: Modify their string, Send it back to the master

- MPI Compile and Run:
  - Compile
    ```shell
      mpirun –np <num_processors> <program>
      mpiexec –np < num_processors> <program> # a synonym
    ```
  - Start processes
    ```shell
      > mpicc hello_mpi.c
      > mpirun –np 4 a.out
      0: We have 4 processors
      0: Hello 1! Processor 1 reporting for duty
      0: Hello 2! Processor 2 reporting for duty
      0: Hello 3! Processor 3 reporting for duty
    ```
  - By default, MPI chooses lowest-latency communication resource available; shared memory in this case
  - MPMD (Multiple Program, Multiple Data) execution
    - `mpirun –np 2 a.out : –np 2 b.out`
      - launches a single parallel application
      - All in the same MPI_COMM_WORLD
      - Ranks 0 and 1 are of instances a.out
      - Ranks 2 and 3 are of instances b.out

- Performance
  - bottleneck is the message passing
  - much longer latency and much less B/W

### MapReduce

MapReduce framework does several things
- Runs tasks in parallel
- Manages communication and data transfers
- Provides redundancy and fault tolerance

#### OverView

- Consists of two functional operations: map and reduce
  - Map: applies a function to an iterable data set
  - Reduce: applies a function to an iterable data set cumulatively
  
- Map
  - Split the input into multiple pieces
  - Pieces are processed as (key, value) pairs
  - Mapper function uses these (key, value) pairs outputs another set of (key, value) pairs
- Reduce
  - Collects the input from the previous map
    - May be on different nodes and require copying
  - Merge-sorts the input
    - So that key-value pairs for a given key are contiguous
  - Reads the input sequentially and splits values into lists of values with the same key
  - Passes this data (keys and lists of values) to your reduce method (in parallel) and concatenates results
- Combine
  - Optional step
  - Could do reduce step for in-memory values after map