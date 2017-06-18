# Topic 3 Multithreading (1)
<div id="author-signature">Leedehai</div>
Friday, April 28, 2017<br>Monday, May 1, 2017

> A *thread* of execution is the smallest sequence of instructions that can be managed by a scheduler. <br>**A thread is a component of a process**, a recipe of independent execution. <br>Multiple threads can exist within one process, being **executed concurrently** and **sharing the process's resources** such as memory space, executable code, variables, etc. while different processes do not share them.<br> There are a lot of ideas in multiprocessing which lend themselves to *multithreading*, another form of *concurrency programming*, where multiple logical control flow overlap or interleave in time.

## 3.1 Overview
- A program does not have to rely on multiprocessing to run multiple functions at the same time. As an alternative, it can create multiple threads with one process.
    > Examples: (1) an online music application may choose to download multiple songs simultaneously, (2) a web browser may choose to load multiple web pages simultaneously using multiple threads.

### 3.1.1 A naïve example
In psychology, [introversion](https://en.wikipedia.org/wiki/Extraversion_and_introversion) is the state of being predominantly interested in one's own mental self. Introverts are typically perceived as more reserved or reflective, but mistaking introversion for shyness is a common error. They can recharge themselves by spending time alone.
>  Introverts prefer solitary to social activities, but do not necessarily fear social encounters.

```C
#include <stdio.h>
#include <pthread.h>

/* thread routine: notice the return type and arugument type */
void *recharge(void *unused) {
    printf("I recharge by spending time alone.\n");
    sleep(1);
    return NULL; /* to statisfy the prototype */
}

int main() {
  pthread_t introverts[6];
  for (size_t i = 0; i < 6; i++)
    /* create 6 independent threads */
    /* All args are pointers */
    pthread_create(&introverts[i], NULL, recharge, NULL);
  for (size_t i = 0; i < 6; i++)
    /* block and wait for the child thread's termination */
    /* The first arg is a TID, the second is a pointer to pointer */
    pthread_join(introverts[i], NULL);
  
  printf("Everyone's recharged!\n");
  return 0;
}
/* takes ~1 sec to run due to multithreading, as opposed to ~6 sec */
```

#### 3.1.1.1 Explanations
- `pthread_t` is essentially an integer - the thread's ID (TID). Similar to `pid_t`.
- The thread routine is a function that **takes in a** `void *` **and returns a** `void *`. In fact, you can pass in a pointer to structure or return a pointer to structure, utilizing pointer's type casting.
- `pthread_create()`'s takes in 4 arguments, all of which are pointers.
    ```C
    int pthread_create(
          pthread_t *pTID, /* TID ptr. */
          const pthread_attr_t *pAttr, /* thread attribute ptr. */
          void *(*pRoutine) (void *), /* thread routine function ptr. */
          void *pArg /* argument ptr. */
        );
    ```
    - The first argument `pTID` points to the variable to store the TID,
    - The second argument `pAttr` points to the structure whose contents are used at to determine attributes for the new thread, such as priority. It can be `NULL`.
    - The third argument `pRoutine` points to the start function of the thread routine, and that function can call other functions.
    - The fourth argument `pArg` points to the argument that is to be passed to the thread routine. It can be a structure if there are more than one actual arguments to be passed, or a `NULL` if none.
    - It returns `0` on success; on error, it returns an error number, and the content of `pTID` is undefined.

- If the master thread ("the mommy thread"), i.e. the thread of the process itself, exists, the entire virtual memory space of the process will be de-allocated, killing every child threads it gave birth to. So we need to wait for the child threads to terminate before letting the master thread terminates: 
    ```C
    int pthread_join(pthread_t TID, void **ppRetval);
    ```
    - The first argument `TID` is the ID of the intended thread. Note that it is not a pointer.
    - The second argument `ppRetval` is a pointer to the pointer to the return value of the thread.
    - If the intended thread has already terminated, then `pthread_join()` returns immediately. Otherwise it **blocks and waits**.
    - The thread specified by `TID` must be **joinable**, i.e. not detached.
        > SIDE NOTE:<br>When a detached thread terminates, its resources are
       automatically de-allocated without the need for
       another thread to join with it. `int pthread_detach(pthread_t TID);` can detach a thread. A detached thread cannot be killed nor reaped by another thread, but a joinable thread can.

    - On success, it returns `0`; on error, it returns an error number, among which are:
        - `EINVAL`: another thread is already waiting to join with this thread, or the intended child thread is not joinable.
        - `ESRCH`: no thread with that TID thread could be found.
        - `EDEADLK`: a deadlock was detected (e.g., two threads tried to join with each other), or the first argument `TID` specifies the calling thread itself.
- Child threads are not obligated to start or finish together or in a predictable order - it is up to the kernel's scheduler (yes, the scheduler can manipulate threads). Once a child thread finishes, `pthread_join()` joins the child thread with the calling thread (in this example, the master thread).
- Failure to join with a thread that is joinable (i.e. undetached), produces a *zombie thread*. Avoid doing this, since zombie threads consume some system resources.

An illustration of creating and joining 2 child threads:
```
    ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐
                     start             finish       a single process
                   ▲──●──────────────────●─ ─ ┐ 
     master thrd.  │   cr   child thrd. 1     │   jn   
    ───────────────┴───┬──────────────────────▼───▲─────────▶ 
                  cr   │      child thrd. 2   jn  │     
                       ▼──────●────────●─ ─ ─ ─ ─ ┘ 
                            start     finish
    └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘
cr: pthread_create()     jn: pthread_join()
```
>SIDE NOTE:<br>ANSI C  doesn't provide native support for threads. But POSIX threads (aka pthread) comes with all standard UNIX and Linux installations of gcc, as concurrency programming requires the cooperation of the kernel.

#### 3.1.1.2 The CPU and memory, thread context switch
- Each process has a thread manager responsible for interleaving the execution time window within the process's execution time.
- Note that the threads created in the example above all belong to **the same process**, hence they reside in the **same virtual memory space**.
- What we are interested in is what happens in the stack and text segments in the virtual memory space.
    - The stack segment: each thread takes a subsegment in the stack, forming their own call stacks. Inside each thread's call stack, there is a stack space reserved to store the CPU register info when that thread is swapped off from the CPU. There is no overlapping in the stack, so automatic variables cannot be inherited by child threads (unlike multiprocessing).
        > The aforementioned space serves as the "switchframe" of the corresponding thread, the difference from a process's switchframe being that a thread's is stored within the stack segment of the process's virtual memory space, while a process's is stored in the kernel space.

    - The text segment: the instruction pointer `rip` points to different instructions, jumping among the master's and child threads' routines.
- All threads share the heap, data, text, .. segments in the virtual memory space. Because of this, **threads within one process can communicate with each other more easily than processes**, who has to turn to pipes and signals.
> In the same process, a thread's access to another thread's **objects in stack** is provided by pointers/references.
>SIDE NOTE:<br>*Preemptive multitasking*: the scheduler can switch the task (process/thread) out from CPU and switch another in, at any time it feels like.<br>*Cooperative multitasking*: the task (process/thread) can only be switched out from CPU when it allows - say, a stage completes - and other tasks have to wait.<br>Cooperative multitasking was the primary scheduling scheme for 16-bit applications employed by Microsoft Windows before Windows 95, and by the classic Mac OS.<br>In computer network protocols, similar mechanisms is applied.

## 3.2 Scheduler's role in thread (un)safety
- When a child thread calls `malloc()`, it draws memory from the heap segment as the master thread does. Note that two threads may both decide to call `malloc()` at the same time, hence the implementation of `malloc()` is transactional, i.e. if it fails halfway, all changes made by it will be rolled back.
- Multiple threads may decide to print something to the console. To prevent garbled output in the console, there is an **exclusive lock** (typically implemented as a boolean) within the **file entry** instance in the system-wide file entry table, and if one thread has taken that lock, another thread cannot take it until the former thread finishes printing and release that lock on the file entry.
    > C's `printf()` is thread-safe; that is, it grabs the exclusive lock on the output file entry before printing and release it after it's done. However, C++'s `ostream::operator<<()` is NOT thread-safe.
- All threads can access the global variables in the data segment. Two threads may both want to alter a global variable's value, so the implementation of the thread routine has to guard against the conflict problem caused by the **unpredictability of the scheduler**. This responsibility falls upon the programmer's shoulder.
An example of the conflict problem:
```
scenaio 1:      ┌────────┐┌────────┐                         ──▶ time   
                │write a:││read a, │                                   
     thread 1   │ a = 1  ││& print │                      => print 1
                └────────┘└────────┘                                   
                                    ┌────────┐┌────────┐
     thread 2                       │write a:││read a, │  => print 2
                                    │ a = 2  ││& print │  
                                    └────────┘└────────┘              OK  
─────────────────────────────────────────────────────────────────────── 
scenaio 2:      ┌────────┐          ┌────────┐                ERRONEOUS  
                │write a:│          │read a, │  
     thread 1   │ a = 1  │          │& print │            => print 2
                └────────┘          └────────┘      
                          ┌────────┐          ┌────────┐                
     thread 2             │write a:│          │read a, │  => print 2
                          │ a = 2  │          │& print │                
                          └────────┘          └────────┘                
```
> CS145, huh? "Two actions *conflict* if they are part of different transactions, involve the same variable, and at least one of them is a write." There is a WW conflict between thread 1 & 2, and in scenario 2, that conflict caused a problem. The second scenario is a bad schedule.

### 3.2.1 An example of problematic multithreading
Suppose there are 6 extroverts and 1 tag-along introvert in an array. The extroverts need to print their greetings but the introvert needs to remain silent.
```C
/* Problematic multithreading */
#include ...

static const char *const names[] = {
  "Albert Chon",         "John Carlo Buenaflor",
  "Jessica Guo",         "Lucas Ege",
  "Sona Allahverdiyeva", "Yun Zhang",
  "Tagalong Introvert Jerry Cain"
};

void *recharge(void *arg) {
    printf("I'm %s. Empowered to meet you.\n", 
           names[*(size_t *)arg]); /* type cast, don't forget */
    return NULL;
}

int main() {
    pthread_t extroverts[6];
    for (size_t i = 0; i < 6; i++) {
        pthread_create(&extroverts[i], NULL, recharge, &i);
    }
    for (size_t i = 0; i < 6; i++) {
        pthread_join(extroverts[i], NULL);
    }

    printf("Everyone's recharged!\n");
    return 0;
}
```
In this program, the array `names` has 7 pointers to C-strings, and the program should have print the first 6 strings one time each (though not necessarily in order). But the actual output is not deterministic: some strings get printed more than once, some strings is not printed, and sometimes the last string, which should not have been printed, gets printed:
```
$ ./confusing-extroverts
I'm John Carlo Buenaflor.  Empowered to meet you.
I'm Jessica Guo.  Empowered to meet you.
I'm Sona Allahverdiyeva.  Empowered to meet you.
I'm Sona Allahverdiyeva.  Empowered to meet you.
I'm Yun Zhang.  Empowered to meet you.
I'm Tagalong Introvert Jerry Cain.  Empowered to meet you.
Everyone's recharged!

$ ./confused-extroverts 
I'm Sona Allahverdiyeva.  Empowered to meet you.
I'm Tagalong Introvert Jerry Cain.  Empowered to meet you.
I'm Tagalong Introvert Jerry Cain.  Empowered to meet you.
I'm Tagalong Introvert Jerry Cain.  Empowered to meet you.
I'm Tagalong Introvert Jerry Cain.  Empowered to meet you.
I'm Tagalong Introvert Jerry Cain.  Empowered to meet you.
Everyone's recharged!

$ ./confused-extroverts 
I'm Tagalong Introvert Jerry Cain.  Empowered to meet you.
I'm Tagalong Introvert Jerry Cain.  Empowered to meet you.
I'm Tagalong Introvert Jerry Cain.  Empowered to meet you.
I'm Tagalong Introvert Jerry Cain.  Empowered to meet you.
I'm Tagalong Introvert Jerry Cain.  Empowered to meet you.
I'm Tagalong Introvert Jerry Cain.  Empowered to meet you.
Everyone's recharged!
```
The bug is:
- Note that recharge references its incoming parameter this time, and that `pthread_create()` accepts the location of the surrounding loop's index variable via its fourth argument. `pthread_create()`'s 4th argument it always passed verbatim as the single argument to the thread routine.
- The problem: The master thread advances `i` **without regard for the fact that `i`'s address was shared with each of the child threads** in its own stack frame.
- At first glance, it's easy to assume that `pthread_create()` captures not just the address of `i`, but the value of `i` itself. That assumption would be incorrect, as it copies the address and that's all.
- The address of `i` (even after it goes out of scope) is constant, but its contents **evolve in parallel with the execution of the child threads**. When dereferencing, `*(size_t *)arg` takes a snapshot of whatever `i` just **happens to be** at the time it's evaluated.
- This is a simple example of what's called a *race condition*.

> Q: can "Albert Chon" be printed more than once?<br> A: No. As the first string, "Albert Chon" can be printed either 0 or 1 time, but not more than that.

> Q: how many times can the `k`-th string (`k` = 0, 1, ..., 6) be printed?<br> A: the `k`-th string can be printed 0 ~ (`k` + 1) times.

Illustration:
```
Scenario 1: OK                                   get 5
                                         ch. 5 ○─●──────────...   t
○ get &i                                  get 4                   o
● dereference                   ch. 4  ○──●────────────────...
                                  get 3                           b
                       ch. 3  ○───●────────────────────...        e
                          get 2                   
               ch. 2  ○───●────────────────────────...            j
                 get 1                                            o
      ch. 1  ○───●──────────────────────────────...               i
        get 0                                                     n
ch.0 ○──●─────────────────────────────────...                     e
                                                                  d
 ───●──────●───────●───────●───────●───────●──────●───────......─────────▶
i = 0      1       2       3       4      5       6(break) master thread
```
```
Scenario 2: ERRONEOUS                           get 5
                                        ch. 5 ○─●──────────...   t
○ get &i                                 get 4                   o
● dereference                  ch. 4  ○─●────────────────...
                                          get 4                  b
                      ch. 3  ○────────────●────────────...       e
                                               get 5                   
               ch. 2  ○────────────────────────●───────...       j
                get 1                                            o 
     ch. 1 ○───●──────────────────────────────...                i
             get 1                                               n   
ch. 0 ○──────●──────────────────────────...                      e
                                                                 d
 ───●───●───────────●───────●───────●──────●──────●───────......────────▶
i = 0   1           2       3       4      5      6(break) master thread
```
```
Scenario 3: ERRONEOUS                get 6
                    ch. 5 ○──────────●───...                     t
○ get &i                              get 6                      o
● dereference  ch. 4  ○───────────────●────...
                                         get 6                   b
           ch. 3  ○──────────────────────●────...                e
                              get 6    
       ch. 2  ○───────────────●──────...                         j
                                                 get 6           o 
   ch. 1  ○──────────────────────────────────────●──...          i
                                  get 6                          n 
ch. 0 ○───────────────────────────●──────...                     e
                                                                 d
 ───●───●───●───●───●───●───●─────────────────────......────────────────▶
i = 0   1   2   3   4   5   6(break)                       master thread
```

To fix this, you can pass in the address to the C-strings themselves, instead of the address of the index `i`.
```C
/* Bug fixed */
#include ...

static const char *const names[] = {
  "Albert Chon",         "John Carlo Buenaflor",
  "Jessica Guo",         "Lucas Ege",
  "Sona Allahverdiyeva", "Yun Zhang",
  "Tagalong Introvert Jerry Cain"
};

void *recharge(void *arg) {
    printf("I'm %s. Empowered to meet you.\n", 
           (const char *)arg); /* type cast, don't forget */
    return NULL;
}

int main() {
    pthread_t extroverts[6];
    for (size_t i = 0; i < 6; i++) {
        pthread_create(&extroverts[i], NULL, recharge, (void *)names[i]);
    }
    for (size_t i = 0; i < 6; i++) {
        pthread_join(extroverts[i], NULL);
    }

    printf("Everyone's recharged!\n");
    return 0;
}
```
> Putting `pthread_join()` in the same loop body as the `pthread_create()` also fixes this bug, but this approach essentially does not utilize the power of multithreading - it is just a sequential program in disguise.

###### EOF