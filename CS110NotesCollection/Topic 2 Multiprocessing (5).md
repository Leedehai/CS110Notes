# Topic 2 Multiprocessing (5)
<div id="author-signature">Leedehai</div>
Monday, April 24, 2017<br>Friday, April 28, 2017

## 2.7 How does multiprocessing works?
> Q1: How does one process behave as if it is the **only** process running in the memory and occupies all the virtual memory space?<br>Q2: The number of processes running on a machine is almost always more than the number of CPU's logical cores. How does the CPU handle them?

### 2.7.1 Virtual-physical memory mapping
- The volume of the virtual memory space for a process is quite large, typically the same size as the physical main memory.
- However, a process does not need that much space. For example, its stack may only occupy several KB in the virtual memory space (several GB).
- Virtual address space is slivered into multiple **fixed-size** parts, termed *page*s. The size of a page is typically a perfect power of 2 (e.g. 256 bytes).
- Actually occupied **pages are mapped into the physical memory**. The memory mapping unit (MMU) hardware inside the CPU is responsible for memory-lookup, with the help of a lookup table, called the *page table*, maintained by the kernel.
    > If a page is not used, it is not mapped and thus does not have an entry on the page table.
- The page table is effectively a big dictionary. Each entry's key is a combination of the PID and the virtual address of the page, and the corresponding value is the physical address of that page (and other info). A process corresponds to multiple entries, the number of which depends on the actual size of memory it occupies.
- The MMU uses the **page table entry** (PTE) and the virtual address's **offset from its page's start**, to find the corresponding physical address.
    > The size of a page is a trade-off between time and space efficiency. If the page size is too small, the page table becomes too large; if the page size is too large, physical memory space is wasted.
- If the process requests additional memory pages (at which time the kernel issues an "page fault" exception), a new entry is set up in the page table, dedicating an available block in the physical memory to the process.
- Through this mechanism, a process may actually occupies a tiny fraction of the physical memory, though its virtual memory space is the same size as the main memory.
- Processes' pages may be interleaved and out-of-order in the physical memory.

A simplified illustration.
```
┌───────────────────────────┐ MMU look-up ┌───────────────────────────┐
│                           │     ┌──────▶│       1207: page 3        │
│       kernel space        │     │       ├───────────────────────────┤
│                           │     │ ┌────▶│       1207: page 0        │
├───────────────────────────┤     │ │     ├───────────────────────────┤
│     (unused, unmapped)    │     │ │     │ (taken by another process)│
├───────────────────────────┤     │ │     ├───────────────────────────┤
│       1207: page 0        │─────│─┘     │                           │
├───────────────────────────┤     │       │            ...            │
│       1207: page 1        │─────│─┐     │                           │
├───────────────────────────┤     │ │     │                           │
│       1207: page 2        │─────│┐│     ├───────────────────────────┤
├───────────────────────────┤     │└│────▶│       1207: page 2        │
│     (unused, unmapped)    │     │ │     ├───────────────────────────┤
├───────────────────────────┤     │ └────▶│       1207: page 1        │
│            ...            │     │       ├───────────────────────────┤
│                           │     │       │ (taken by another process)│
├───────────────────────────┤     │       ├───────────────────────────┤
│       1207: page 3        │─────┘       │                           │
├─────────────────0x6000acff┤             │            ...            │
│       1207: page 4        │───────┐     │                           │
├─────────────────0x6000ac00┤       │     │                           │
│            ...            │       │     ├─────────────────0x2cccfaff┤
│                           │       └────▶│       1207: page 4        │
└───────────────────────────┘             └─────────────────0x2cccfa00┘   
virtual memory of proc. 1207                    physical memory
```

### 2.7.2 Memory manager and lazy mapping
The memory manager is a component of a OS kernel that is responsible for virtual-physical memory mapping.
#### 2.7.2.1 The layout of an executable
Illustrated below is the general layout of an executable file, the final output of a compiling system.
> SIDE NOTE:<br>There are many executable file format, such as Unix's 32-bit ELF and 64-bit ELF, macOS and iOS's Mach-O, and Window's PE.
```
┌─────────────────────────────┐
│          bss info           │ <- info of uninitialized global variables,
│ (uninitialized global var.) │    like their sizes
├ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┤
│           rodata            │ <- read-only global variables, i.e. global
│     (const global var.)     │    variables declared with "const".
├ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┤
│            data             │ <- other global variables.
│        (global var.)        │
├─────────────────────────────┤
│                             │
│                             │
│                             │ <- binary instructions stored here.
│            text             │
│    (binary instructions)    │
│                             │
│                             │
│                             │
└─────────────────────────────┘file start
```
Note that the "bss" segment in the executable file does not store the uninitialized global variables themselves, only their basic information (e.g. their sizes). That being said, uninitialized global variables of a few primitive types in C/C++, like `int` and `bool`, can be filled with `0` during loading time.

#### 2.7.2.2 Memory management
- When process called `execvp()` (or similar ones), the *loader* begins to load the intended program's executable into the process's virtual memory space.
    - The loader creates segments in the virtual memory space for text, data, and rodata, according to their size information, but does not actually write bits into them.
    - The loader creates a bss segment in the virtual memory space according to the size information stored in the bss segment of the executable file.
    - The loader creates the stack in the virtual memory space. The initial stack size is the loader's "best guess". Similarly, the loader creates the heap in the virtual memory space.
    > The stack is meant to store automatic variables. The heap is meant to store static variables and memory chunks that is allocated dynamically.

    - Note that the loader only creates the segments in the virtual memory space; they are not necessarily back by the physical memory.
- Up to this point, the virtual memory space is **not** mapped to the physical memory.
- When execution begins, the instruction counter points to the starting virtual address of the text segment, and then the MMU maps the virtual page around that virtual address to the physical memory, and then instructions are fetched from the executable file on disk (or typically, cache) to fill that physical page.
    > Indeed, only **part of the instructions are actually written into the physical memory**. For example, if a program is inside a while-true loop, then typically only one or two pages that stores the instructions in that loop are written to the physical memory. If the system determines it needs more, it will fetch more instructions from the executable file accordingly.
- Likewise, only when the program wants to access a global variable does the virtual page on which global variable reside be mapped into the physical memory. It is the same case for the stack and heap.
> This conservative mapping mechanism is termed *lazy mapping*, which is meant to save physical memory space and thus support multiprocessing, with the cost of time efficiency.
- The physical page continues to exist until the OS determines the physical memory is nearly full, at which point the OS evicts pages that it deemed useless (usually according to the page's time stamp). The evicted pages are gone or transferred into cache or disk for future use. So even the eviction procedure is "lazy".


### 2.7.3 Scheduler and context switch
The scheduler is a component of the kernel that is responsible for context switching, i.e. interleaving the execution flow of processes.
#### 2.7.3.1 Context switch
- Context switch is a procedure that the kernel interleaves the execution flow of multiple processes.
- When performing a context switch, 
    - **CPU - memory**: the kernel stores all the info on the CPU registers critical to the process currently running on the CPU in to a data structure, *switchframe* (or referred to as *process control block*, PCB), in the kernel memory, and
    - **memory - CPU**: load another process's info from the switchframe on memory to CPU registers, so the process continues to run **as if it has never left**.
    > SIDE NOTE:<br>The CPU registers include `rip` (instruction counter), `rsp` (the top of the stack), `rax` (return value), `rdi` (first argument) .. on a x86 CPU.
```
                                               
   process A             OS             process B
       ┃ :                                      
       ┃ #101 (instruction)                               │
       ┃ #102                                             ▼ time
   ─ ─ ▼ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─      tick
              save A's reg. value to memory  
             load B's reg. value from memory             
   ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┰ ─ ─ 
                                               ┃
                                               :
                                               ┃
   ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ▼ ─ ─      tick
              save B's reg. value to memory
             load A's reg. value from memory             
   ─ ─ ┰ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
       ┃ #103                                      
       ┃ #104 
       : :                                   
   ─ ─ ▼ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─      tick
```
#### 2.7.3.2 Three classes of processes: the running, ready, and blocked
- Processes that are alive (i.e. initiated and not terminated) are categorized by the kernel into three classes:
    - The running, processes that are actively executing on the CPU. Each CPU core can handle at most **one process** at a time.
    - The ready, processes that are temporarily off the CPU but is ready to be reloaded back on the CPU. A process typically spends the majority of its lifetime here.
    - The blocked, processes that are blocked (i.e. suspended), e.g. by `sleep()`, `waitpid()`, `sigsuspend()`, other signals, or waiting for a keystroke with an empty buffer.
- One CPU core can handle one process at a time. If the time window expires, the process is transferred to the "ready" class and another process from the "ready" class is loaded in. If the process is blocked, it is immediately demoted to the "blocked" class.
- Processes in the "ready" class are stored in what is known as a *ready queue*, each node of which is a snapshot (switchframe) of a process's register info taken at the instant when the process is switched out.
- Processes in the "blocked" class are stored in what is effectively a set, each node of which is also a snapshot of a process's register info. A blocked process stays in the "blocked" class until the event that triggers its revival happens, at which time the kernel would lift the process's switchframe out of the set and put it to the ready queue.
    > SIDE NOTE:<br> Therefore, you cannot rely on a `sleep()` call to maintain a precise timer, as copying the snapshot from the set to the ready queue and waiting in the queue consume extra time (i.e. CPU periods).

A process's lifetime illustrated in a state diagram:
```
             ┌───────────────┐                    ┌────────────┐
             │    running    │ exit(), SIGKILL... │ terminated │
         ┌───┤ (one per core)├────────────────────▶  (zombie)  │ => reap
         │   └────┬─────▲────┘                    └────────────┘
         │  switch│     │switch                          
         │     out│     │in                          
  sleep()│   ┌────▼─────┴────┐                     
waitpid()│   │     ready     │ admitted&initiated ┌────────────┐
  SIGSTOP│   │ (ready queue) ◀────────────────────┤  created   │ <= create
      ...│   └───────▲───────┘                    └────────────┘
         │           │                            
         │           │ event of interest                           
         │   ┌───────┴───────┐                     
         │   │    blocked    │                     
         └───▶ (blocked set) │                     
             └───────────────┘                     
```
###### EOF