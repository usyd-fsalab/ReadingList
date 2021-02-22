# SCALENE: Scripting Language Aware Profiling for Python

Scripting language suffer from a multitude of problems including **order of magnitude overheads, information reported at too coarse of granularity, or fail in threading.** NOTE: profiling is essentially the analysis of the programming language that is dynamic and measures space, complexity, calls of instructions, frequency of calls etc. The issue here and as mentioned in the other paper is that **there is a divide between the scripting language and libraries written in the compiled languages**

**The SCALENE is a scripting language aware profiler for Python, it does sampling, inference and attribute execution time memory usage to Python.**

The additions are that:

- it includes a novel sampling memory allocator that reports a more in depth line level memory consumption which is low overhead --> **reduces footprints and helps identify leaks**
- copy-volume: **new metric to help eliminate copy costs on the Python/library(compiled) boundary.** --> helps improve performance.
- works for single and multithreaded Python code and provides detailed information at line granularity.

## What is the problem

Scripting language suffer from a multitude of problems including **order of magnitude overheads, information reported at too coarse of granularity, or fail in threading**

Moreover what the issue is for each is that:

- scripting languages share many implementation characteristics such as **un-optimised bytecode interpreters**  
- **inefficient garbage collectors** --> (like **reference counting, which cannot reclaim cycles** leaving memory wasted and since objects in these scripting languages contain a lot of metadata their size is quite obnoxious). 
- The profilers are essentially ports of profilers for other systems
- As well as **limited support for signals.** The scripting languages have very inefficient ways of dealing with signals, in Python, the signals are delayed until the virtual machine regains controls (usually after each opcode). This essentially means no signals are delivered and hence no samples are collected during the time Python makes library calls. 

## Why is this a problem

Well the reason why these issues are worth discussing is to notice that the combination of all of these issues result programs to run many magnitudes slower than their system language counterparts (C etc).

Moreover **profilers are essentially ports of profilers for other systems languages**  which is not specific to scripting languages **and cannot help to identify the issues discussed above.** As discussed because of the space requirements of objects in these scripting languages (Python) >= 24-28 bytes in combination with poor memory management such as reference counters **which cannot reclaim cycles** leads to a huge waste of memory where **current profilers struggle in detailing areas wherein we can optimise.**

### Motivation for better system

Since these scripting languages are so inefficient then the key way to optimise **involves moving code down into the native libraries.** Hence, developers need to know where the bottlenecks are in their code and where they should be using native library calls in order to optimise. **recall we are assuming that we are dealing with optimising scripting language sections and not native library calls.** Also, the developers need a precise profiler in order to limit unnecessary memory consumption by:

- avoiding **accidental instantiation of lazily generated objects**
- moving scripting code in libraries
- as well as identify leaks.
- Need to identify and eliminate implicit copying.

We need to also determine who uses python and would benefit from this optimisation, Google, Dropbox, Facebook, Instagram, Spotify all use it.

### Example

![1612852634739](img/1612852634739.png)

Looking here we can see that all these scripting languages lack threads or serialise them with a global interpreter lock and place **severe limits on signal delivery.**

## Why is this solution novel

![1612852718814](img/1612852718814.png)

In comparison to other Python profilers Scalene has an extremely high efficiency, profiles memory consumption, does not require the source to be modified and deals with threaded implementations.

- Separate Python/C accounting of time and space, instead of giving a basic response, the SCALENE will showcase whether it stems from Python or native code. It helps show whether they need to spend time to optimise or not, since we can only really optimise the non native code (not exactly).
- Fine grained tracking of memory use over time by using a **novel sampling memory allocator**, to profile memory usage efficiently **for per-line memory profiles in the form of sparklines**, **indicate trends of consumptions.**
- **Copy volume** - reports copy volume in Mb/s for each line of code which makes it easier to spot inadvertent copying, or crossing into the Python/library boundary e.g. numpy to python arrays without knowing.

It has a 26-53% overhead for full profiling which is more efficient than other profilers (check). **Other profilers are more generic to system languages and hence to not provide meaningful insights.**

## How is this done

`scalene app.py` 

by default it generates a **profile when the program finishes.**

**SCALENE breaks out CPU usage by:**

- whether it is attributable to interpreted or native code.
- Its sampling memory allocated which replaces library default by **library interposition (LD_PRELOAD)** allows it to report line granularity net memory consumption to Python or native code. Easily identifiable memory graphs.

e.g. below is using line_profiler: interesting to note, is that the developer of this profiler notes that they suggested forking it to use **rotating trees rather than dicitionaries and extension objects to store entries.**

![1612854279895](img/1612854279895.png)

instead using SCALENE it shows that we have an odd memory usage over time. This follows a sawtooth pattern and may suggest that there is an unnecessary copy. It copies to a temporary which is allocated and then discarded. This has a call to `np.array` which is shown below.

![1612854403922](img/1612854403922.png)

Does not depend on any modifications to CPython interpreter and this means that **SCALENE could be portable.**

The goal of SCALENE is to attribute execution time separately so that developers can identify which code they can optimise. Whether it is Python to native code. **But since the main thread is essentially blocking to all signals the typical solution of handle signals -> traverse stack -> distinguish where code was called is unable to be done.**

![1612918494176](img/1612918494176.png)

Instead SCALENE utilises an extremely interesting trick, because since we know that signals are delayed or blocked during this period, we can consider that **any delay in signal delivery corresponds to time spent executing OUTSIDE OF THE INTERPRETER.** e.g. if we received the signal immediately then that means we are dealing with the interpreter, but if it was delayed then it is due to being outside of the interpreter.

SCALENE tracks this time delayed using `time.process_time()` or `time.perf_counter()` to record the last time it received a CPU interrupt.

### Scalene Algorithm of Timing

To assign the time to either Python or C:

- SCALENE receives a signal
- Walks the Python stack until it reaches code being profiled (outside of libraries, or python interpreter) and times this piece of code.
- Maintains two counters for **every line of code bring profiled. (One for C and Python).**
- Now since we know that during library calls the main thread does not respond to signals then when a line is interrupted by a signal **SCALENE increments the python counter by q (the timing interval) and increments the C counter by** $T - q$. e.g. Keeps getting interrupted after every 0.01 second interval means that its executing primarily Python code which is still an estimation (no description as to why other than below).

By updating both counters they yield an unbiased estimator, where the estimates are equivalent to the actual execution times.

The way their approach works is as such:

- say a line spends 100% of its time in the Python interpreter then when a signal occurs **it will be immediately delivered, e.g. T = q**
- Thus the C counter will yield 0 and the Python counter will have value T = q.
- However when the line is primarily executed in C time then we have 100% of time there and hence the ratio of C code over C + python is:
  - $\frac{T - q}{(T - q) + q}$ as we reach $\lim_{T \rightarrow \infty}$ then $\frac{T - q}{T} = 1$
  - A line that takes one second executing C code would have $\frac{1 - 0.01}{1}$ or 99% of samples as native code.

They modelled a problem set to quantify how quickly the formula converges depending on the ratio of T and q.

![1612921507613](img/1612921507613.png)

### Dealing with Measuring Thread Time

To **handle attributing the execution time of threads SCALENE uses EXISTING python features:**

- Monkey patching: refers to redefinition of functions at runtime (which is a feature of most scripting languages). This is used to ensure that signals are always received by the main thread **even when it is blocking**. It maintains a status flag for every thread, **all initially executing**, then when SCALENE intercepts  a call before it blocks the thread it will set it as sleeping and reversed when it resumes. **Only executing threads will be attributed.**
- Thread Enumeration: when main thread receives a signal, SCALENE collects a list of all running threads.
- Stack inspection: then obtains the Python stack frame from **each thread** using a build in **sys** method and walks the stack to find the appropriate line of code to start timing execution.
- Bytecode disassembly: **disassembles using the** `dis` **module to determine how time spent in Python vs C code** e.g. it does this based on the fact that whenever Python invokes an external function it does so via the bytecode operation `CALL_` and it builds a map of all these bytecodes to determine whether for each running thread to determine if its executing a CALL instruction..

All  of which are available in most other scripting languages.(*).

### Memory Usage

SCALENE intercepts all memory allocation related calls (`malloc`, `free`, etc) via its own replacement memory allocator. NOTE: interesting implementation detail of Python is that for allocations of 512 bytes or less Python will use on its own internal memory allocator and maintains its own  freelist but if the environment variable PYTHONMALLOC is changed to `malloc` then it will use `malloc` instead.

Their novel sampling memory allocator was created due to the large overheads of using a standard system allocator, slowed their running time by around 80%. Built their C++ allocator using Heap Layers infrastructure and similar to the Python allocated but with a few additions.

- We can't use the default allocator because of performance overhead and we can't extract allocator from Python source code (due to it being implemented on top of `malloc`). 

There is a need to have this feature because library interposition will possibly not intercept all object allocations, hence the allocator should be able to quickly identify a foreign object and discard them **so there is no free on foreign objects (objects not allocated by malloc).** EXTRA: It does this by checking if objects are in a specific contiguous virtual memory range, and determining whether the magic number allocated to them is valid, or not aligned to 4K boundaries, if they are not then it is treated as invalid. 

### Sampling

The sampling memory allocator maintains a count of all memory allocations and frees, once either crosses a threshold then **a signal is sent to the Python process.**  This is set to prime number > 1MB (reason associated to stride behaviour interfering with sampling).

**Call stack Sampling:**

To track whether the allocated objects were allocated by Python or native code SCALENE uses stack sampling by climbing the stack.  

How this works:

- Sets a sampling rate of the frequency of allocation samples (13x).
- Whenever 1MB (interval for sampling) / 13 allocations have been crossed it climbs the stack
- To distinguish if it is PYTHON allocation: it searches for references which begin with `Py_` or `_Py` which is domain specific to Python allocated objects (another limiting factor of portability) and increments Python counter for allocations.
- For native code it keeps walking the stack until it reaches a maximum number of frames and if it cannot find the reference starting with `Py_` or `Py_` then it classifies as C code and increments it.

**Managing Signals:**

- Python does not queue signals --> so we need a separate channel or else signals will be lost. The way this is done is via a temporary file which uses the process-id as the suffix. SCALENE appends information about allocation or frees etc.
- When the signal handler of SCALENE is triggered it reads the temporary file and attributes allocations or frees the currently executing line of code in every frame. It also tracks the current memory footprint which it uses **to report maximum memory consumption as well as to generate sparklines for memory allocation trends.**

### Memory trends

Reports net memory consumption per line, and reports memory usage over time in the form of sparklines, for the program as a whole and each individual line. It adds the current footprint (updated on each allocation and free) to an ordered array of samples for each line. The median of this sample array is chosen to smooth older footprint trends while maintaining fidelity for newer footprints.

### Copy Volume

SCALENE also report the copy volume for each line, which is done by sampling, interposes `memcpy` and triggers a signal when a threshold number of bytes have been copied. The copying is usually preceded by an allocation of the same size then a deallocation.

## Results

Results and evaluation was done on a MacBook Pro. This below table investigates the portability of this SCALENE profiler to other languages.

![1612972589466](img/1612972589466.png)

In comparison this was benchmarked using `bm_mdp` simulating battles in games, and compared to multiple other porfilers:

![1612972675561](img/1612972675561.png)

SCALENE full profiling in general has a 26% - 53% overhead.

Another example of use:

![1612972794378](img/1612972794378.png)

From the line_profiler we can clearly see that the issue is due to line 15 but the problem is like with many other profilers **that we cannot analyse why easily.** Using the SCALENE profile it shows that the majority of this code is in Python interpreter and the the memory usage over time is approximately 81%, which hints at a lot of allocation and deallocation of memory, specifically with num and fact. Hence, the idea would be to keep the numbers of num and fact smaller by maintaining a ratio variable rather than using num and fact separately which drops execution time by 1000 times. 

### Related Work

| Perl                                                                                                 | TCL                                          | Python                                                                                     | Lua                                               | PHP                                                                                | R                                               | Ruby                                                                                 |
| ---------------------------------------------------------------------------------------------------- | -------------------------------------------- | ------------------------------------------------------------------------------------------ | ------------------------------------------------- | ---------------------------------------------------------------------------------- | ----------------------------------------------- | ------------------------------------------------------------------------------------ |
| Uses abstract-syntax tree based interpreter                                                          | stack based bytecode interpreter             | Stack based bytecode interpreter                                                           | Register based interpreter.                       | Similar to register based interpreter                                              | AST- based interpreter and bytecode interpreter | Originally AST but now bytecode interpreter                                          |
| Reference counting garbage collector                                                                 | Reference counting garbage collector         | reference counting garbage collection (can use gc module)                                  | Relies on incremental garbage collection          | Reference counting garbage collection                                              | Relies on mark-sweep garbage collector          | Incremental GC (in 2.1)                                                              |
| Interpreter threads                                                                                  | Interpreter threads (as an extension)        | Only one thread can execute in the interpreter at a time using the global interpreter lock | Has cooperative (non-preemptive threads).         | Not thread safe but can enable ZTS                                                 | Single threaded                                 | It serialises the threads through a global interpreter lock.                         |
| Signal delivery is delayed until interpreter enters a safe which is usually between operation codes. | Not supported (available through extensions) | Signals are only delivered to the main thread and delayed until VM regains control.        | Signals are delayed until the VM regains control. | Signal are delayed much longer only when the VM reaches a jump or call instruction | No support                                      | Signals are delivered only to the main thread and delayed until interpreter returns. |

Alternatives to SCALENE for Python profiling:

- Only one is a CPU profiler, and most are less efficient, `yappi` does do CPU profiling and has different modes but does not do sampling so is much more inefficient.
- They either have function or line granularity.

### Limitations and Future Work

Argues portability but suffers from memory allocation redirection, and bytecode disassembly for other languages.

To distinguish if it is PYTHON allocation: it searches for references which begin with `Py_` or `_Py` which is domain specific to Python allocated objects (another limiting factor of portability) 

There seems to be an application of this to other languages or more so to develop a general purpose profiler that operates across multiple different scripting languages providing features offered in SCALENE with the possibility of automatic optimisations. There is no scripting language aware profiler for the other languages and are missing features that SCALENE incorporates.

- Ruby: profilers `stackprof` does not integrate with CPU sampling, nor any scripting language aware profiling - copy volume, tracking memory usage over time, and can't simultaneously perform CPU profiling and memory tracking.
- R: has a promising profiler that has feature specific profiling which attributes costs to specific language features. Would need something like a copy volume implementation to detect unnecessary allocation and deallocations.
