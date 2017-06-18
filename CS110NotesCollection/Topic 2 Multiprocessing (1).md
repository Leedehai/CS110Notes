# Topic 2 Multiprocessing (1)
<div id="author-signature">Leedehai</div>
Friday, April 14, 2017

> A *process* is an instance of a computer program that is being executed. *Multitasking* is a method to allow multiple processes to share processors (CPUs) and other system resources. <br> By the way, a *thread* of execution is the smallest sequence of instructions that can be managed independently by the OS. Multiple threads can exist within one process, executing *concurrently* and sharing the executable code, variable values, and resources such as memory, while different processes do not share them.

## 2.1 Mitosis: `fork()`
### 2.1.1 Overview
- The system call `fork()` clones the calling process, create an **identical copy** of it in memory, and schedule it **as if this copy of the original were running all along**. All segments (data, bss, init, stack, heap, text) are faithfully replicated, and all file descriptors are copied to the clone.
    > "as if this copy of the original were running all along" means the program counter points to the **same instruction** in the clone as in the original, and all data that exists in the original before the moment of `fork()` are copied.

    > "all file descriptors are copied to the clone" means that a copied file descriptor represents the **same file entry** in the system-wide file entry table as the original one does, and the two file descriptors equal in value.

    > Obviously, `fork()` is a deep copy, not a shallow copy.

    > On a modern OS, `fork()` may adopt a copy-on-write strategy (lazy copy).

- The only difference: `fork()`'s **return value** in the new process (the child) is **0**, and `fork()`'s return value in the spawning process (the parent) is the **child's process id** on success (`-1` on failure). The return value can be used to dispatch each of the two processes in a different routine.
- As a result, if in a program's code it executes `fork()` once, the output of the whole program is really the output of two processes. If twice, then four processes. The output's **order is unpredictable** in that it is dependent on the kernel's scheduler, and they might even run in parallel on a multicore machine.
- A nice diagram to illustrate:
```
                                                            children
                                                           ┌────────▶
                        child                              │       
                ┌────────────────▶                  ┌──────┴────────▶
                │                                   │              
                │                                   │      ┌────────▶
  parent        │      parent             parent    │      │       
────────────────┴────────────────▶     ─────────────┴──────┴────────▶
              fork()                             fork()   fork()
                          time ──▶                           time ──▶
     the program forks once                 the program forks twice
```
> The system call `fork()` is not a nobody. On Unix, almost every process is created by `fork()`.
### 2.1.2 An example
```C
#include <stdbool.h>      // for bool
#include <stdio.h>        // for printf
#include <unistd.h>       // for fork, getpid, getppid

int main(int argc, char *argv[]) {
    printf("Greetings from process pid = %d (parent pid = %d)\n", 
            getpid(), getppid());

    pid_t pid = fork();

    printf("Bye bye from process pid = %d (parent pid = %d)\n", 
           getpid(), getppid());

    return 0;
}
```
Note: system calls `getpid()` and `getppid()` gets the current process's process ID and its parent's process ID, respectively. The return type is essentially an integer. The parent ID is usually smaller than the child's ID.

### 2.1.3 Explosive `fork()` graphs
Be aware of `fork()`'s spawning power. It may be used in a computer virus.
```C
#include <unistd.h>     // for fork
#include <stdio.h>      // for printf, etc
#include <string.h>     // for strlen

static const char const *kTrail = "abcd";
static const int kForkFail = 1;

int main(int argc, char *argv[]) {
  printf("Let the forking begin.\n");
  size_t trailLength = strlen(kTrail);
  for (size_t i = 0; i < trailLength; i++) {
    printf("%c\n", kTrail[i]);
    pid_t pid = fork();
  }
  printf("\n");
  return 0;
}
```
Ouptut: 1 `a`, 2 `b`'s, 4 `c`'s, 8 `d`'s，16 running processes.
> Note that the terminal's prompt may be printed in the middle of the output letter string, because when the parent process (the one that is initiated directly from the terminal) returns, it causes the terminal to print a prompt line, paying no attention to the other running child processes.

Illustration (a binary tree, no surprise):
```
                                             ┌─────────▶ 
                                     ┌──'d'──┴─────────▶ 
                                     │       ┌─────────▶ 
                             ┌──'c'──┴──'d'──┴─────────▶ 
                             │               ┌─────────▶        
                             │       ┌──'d'──┴─────────▶ 
                             │       │       ┌─────────▶ 
                     ┌──'b'──┴──'c'──┴──'d'──┴─────────▶   
                     │                       ┌─────────▶   
                     │               ┌──'d'──┴─────────▶ 
                     │               │       ┌─────────▶  
                     │       ┌──'c'──┴──'d'──┴─────────▶  
                     │       │               ┌─────────▶ 
                     │       │       ┌──'d'──┴─────────▶  
                     │       │       │       ┌─────────▶ 
        ────────'a'──┴──'b'──┴──'c'──┴──'d'──┴─────────▶
```
Note that this illustration doesn't account for the kernel's scheduler, so these processes seem to be running in parallel - in reality, it might not be the case.

## 2.2 Block and wait: `waitpid()`
### 2.2.1 An overview
- This system call instructs a process to block until another process changes its state. It is a way to **synchronize** the parent and child process.
- `pid_t waitpid(pid_t pid, int *status, int options)`
- The first argument specifies the wait set. If it is positive, it indicates the process ID of the child process. If it is `-1`, the system call waits for any child process. If it is other negatives, then the absolute value of this negative indicates a process group's ID.
  > Process group: every process belongs to exactly one process group, which is identified by a positive integer process group ID. By default, a child process belongs to the same process group as its parent. You may examine a process's pgid by system call `getpgid()`. You may also set a process's pgid via system call `int setpgid(pid_t pid, pid_t pgid);` (you may see [here](http://www.tutorialspoint.com/unix_system_calls/setpgid.htm) for more).
- The second argument is the address of an integer where child termination status information can be written in (it can be `NULL` if we don't care to get that status). You may turn to [this page](http://www.tutorialspoint.com/unix_system_calls/waitpid.htm) for more info on the status integer.
- The third argument is an option flag. If it is `0`, then `waitpid()` should only return when a child exits. If it is `WNOHANG`, then `waitpid()` will immediately return `0` without blocking if there is no undiscovered terminated child. For more complicated options, you can see [here](http://www.tutorialspoint.com/unix_system_calls/waitpid.htm).
- The return value is the process ID of the child process that changes state. If `waitpid()` was called but there were no such child process(es) as indicated by the first argument to wait on, or if it encounters an error, it will returns `-1` and automatically sets `errno` to indicate the reason (setting `errno` to `ECHILD` indicates the former case).
  > A number of other system calls also use the global variable `errno` to indicate the reason of failure. `errno` always store the error number set by the most recent failed system call.

### 2.2.2 An example
```C
#include <stdbool.h>  // for bool
#include <unistd.h>   // for fork
#include <stdio.h>    // for printf, etc
#include <sys/wait.h> // for waitpid
#include <time.h>     // for time

int main(int argc, char *argv[]) {
  pid_t pid = fork();
  bool isParent = (pid != 0);

  /* force exactly one of the two to sleep */
  if ((random() % 2 == 0) == isParent) sleep(1); 

  /* parent waits on the child to exit; child waits on no one */
  if (isParent) waitpid(pid, NULL, 0);
  printf("I am printed from the %s).\n", isParent  ? "parent" : "child");
  return 0;
}
```
The program above seduces one of the two processes (parent and child) to sleep for 1 second, using a coin-toss. This allows the other process to have time to print.

In this program, the parent process waits on the child to exit before it allows itself to exit (akin to a parent not being able to rest until he or she knows the child has).

Note that final `printf()` gets executed twice. **The child is always the first to execute it**, however, because the parent is blocked by `waitpid()` until the child exits.

### 2.2.3 A more complicated example: divert the routine
You can use the return value of `fork()` and/or the output status integer to implement different routines for different cases.
```C
#include <unistd.h>
#include <stdio.h>
#include <sys/wait.h>
#include <errno.h>

int main(int argc, char *argv[]) {
  pid_t pid = fork();
  if (pid == 0) { /* child's routine */
    printf("I am the child, and the parent will wait up for me.\n");
    return 110; // return something, a contrived number here
  }
  else if (pid != 0) { /* parent's routine */
    int status;
    pid = waitpid(pid, &status, 0)
    if (pid == -1 && errno = ECHILD) {
            printf("The specified pid does not exist");
    }
    if (WIFEXITED(status)) {
      printf("Child exited with status %d.\n", WEXITSTATUS(status));
    }
    else {
      printf("Child terminated abnormally.\n");
    }
    return 0;
  }
}
```
Here `status` is populated by the `waitpid()`. `WIFEXITED(status)` returns `true` if the child process exits normally by examining the higher bits of `ststus`; `WEXITSTATUS(status)` returns the least significant 16-8 bits of the return value of the child process (`110` here) if the child exits normally.

### 2.2.4 Some loose ends
Q: What if the child process runs so quick that it completes before the parent process reaches the instruction of `waitpid()`? What will `waitpid()` return?

A: In this case, `waitpid()` will still return normally (and immediately without waiting), indicating the child process was terminated. Though the child process completes before `waitpid()` gets executing, the cue of the child's status change is still there to be read.
> In fact, a child that terminates, but has not been waited for becomes a *zombie*. The kernel maintains a minimal set of information about zombie processes (PID, termination status, resource usage information) in order to allow the parent to later obtain information about the child. Completely de-allocating the resources occupied by a terminated process ("zombie") is termed *reaping*.
```
                                .....            
                               O O  /            
                              /<   /             
               ___ __________/_#__=o                    ,
              /(- /(\_\________   \                     ',' ,  
              \ ) \ )_      \o     \                      ', ', 
              /|\ /|\       |'     |                        |  |
                            /o   __\                        |  |
                           / '     |                       ,' ,'
                         /_/\______|                      ,' ,'
                        (   _(    <           / ~-.___ .-'  ,' 
                         \    \    \        /   .______.- ~              
                           \____\___\      /   /'      
                           ____\_\___\      \./      
                         |___ |_______|.. .
                        
                          Reap a zombie.   
```
***
Q: If a child process terminates (and therefore becomes a "zombie"), will `waitpid()` reap that zombie?

A: Yes. Performing a wait allows the system to release the resources associated with the child.
***
Q: Is it possible that the child process begins after the parent has already stepped into the `waitpid()` call?

A: Entirely possible. The kernel's scheduler is responsible for determining the temporal interleaving of the instructions of different processes. If the scheduler was designed so badly that it won't perform a context switch to the child process until the parent's `waitpid()` completes, then the parent is trapped in its `waitpid()` call forever, since the child process never has a chance to complete.
***
Q: What is a child process's "status change"?

A: The running/stopped child terminated (either normally or by signal); the running child was stopped by a signal (e.g. by signal `SIGTRAP` or `SIGTSTP`, `SIGSTOP`); a stopped child was prompted to continue by a signal (e.g. by `SIGCONT`).
***
Q: What if a parent process terminates without waiting for its child process to complete, and terminates before the child process?

A: Then the child process is regarded as an *orphan process*. A special process, typically [init](https://en.wikipedia.org/wiki/Init) (though it depends on the kernel's implementation), will adopt it as child and waits for it to complete and then reap it. This operation is called re-parenting and occurs automatically.

> SIDE NOTE:<br>On Unix-like operating systems, every process except process 0 (the *swapper*, or *idle task*), is created when another process executes the `fork()` system call. Every process (except process 0) has one parent process, but can have many child processes. Process 0 is a special process that is created when the system boots; its child process (process 1), known as *init*, is the ancestor of every other process.

## 2.3 Reap the zombies (KOB)
- On Unix, if a process terminates, it becomes a *zombie*, which means it still consumes some system resources (e.g. a slot in the kernel's process table). Releasing the resources occupied by a zombie process is termed *reaping*.
- System call `waitpid()` can reap zombies. If zombie child is not reaped, it will **continue to occupy system resources** until the parent process itself terminates, at which point the kernel arranges for [init](https://en.wikipedia.org/wiki/Init) process to reap the zombies. Long-running processes should always reap their own zombie children.
  > Another similar system call, `wait()`, has the same effect.
- "Block and wait": `waitpid()` and `wait()` block the calling process while waiting on the child process(es).
- In the examples below, `exit()` is a system call that terminates a process, and uses its argument as the exit status returned to the parent process, and sends a `SIGCHLD` signal to the parent process.
### 2.3.1 Reap as they exit
A child process is reaped as soon as it terminates.
```C
#include <unistd.h>     // for fork
#include <stdio.h>      // for printf
#include <sys/wait.h>   // for waitpid
#include <errno.h>      // for errno, ECHILD
#include "exit-utils.h"

static const int kNumChildren = 8;
static const int kForkFail = 1;
static const int kWaitFail = 2;

int main(int argc, char *argv[]) {
  for (size_t i = 0; i < kNumChildren; i++) {
    pid_t pid = fork();
    exitIf(pid == -1, kForkFail, stderr, "Fork function failed.\n");
    if (pid == 0) exit(110 + i); /* a contrived return value */
  }

  while (true) { /* keeps reaping until there's no zombies left */
    int status;
    pid_t pid = waitpid(-1, &status, 0); /* waits for any child process */

    if (pid == -1) break; /* if faild or no child left, then break */
    if (WIFEXITED(status)) {
      printf("Child %d exited: status %d\n", pid, WEXITSTATUS(status));
    }
    else {
      printf("Child %d exited abnormally.\n", pid);
    }
  }
  exitUnless(errno == ECHILD, kWaitFail, stderr, "waitpid failed.\n");
  return 0;
}
```
### 2.3.2 Reap in fork order
Child processes are reaped in the same order as they are created.
```C
#include <unistd.h>     // for fork
#include <stdio.h>      // for printf
#include <sys/wait.h>   // for waitpid
#include <errno.h>      // for errno, ECHILD
#include "exit-utils.h" 

static const int N = 8;
exitIf(pid == -1, kForkFail, stderr, "Fork function failed.\n");

int main(int argc, char *argv[]) {
  pid_t children[N];
  for (size_t i = 0; i < N; i++) {
    children[i] = fork();
    if (children[i] == 0) exit(110 + i); /* a contrived return value */
  }

  for (size_t i = 0; i < N; i++) { /* wait for each child in order */
    int status;
    pid_t pid = waitpid(children[i], &status, 0); /* waits in order */

    if (WIFEXITED(status)) {
      printf("Child %d exited: status %d\n", pid, WEXITSTATUS(status));
    }
    else {
      printf("Child %d exited abnormally.\n", pid);
    }
  }
  return 0;
}
```

### 2.3.3 Wait a minute.. how many children were ever created in these two examples?
The short answer: in each example, there are 8 child processes. Each example will reap 8 processes and will print 8 lines of text in the terminal.
> There are 9 processes in total for each example, if you count the original parent in.

Q: Ehh.. you said it would be a binary tree! Why weren't there 2<sup>8</sup>-1 = 255 child processes ever created?

A: Note the code block in these examples:
```C
for (size_t i = 0; i < kNumChildren; i++) {
    pid_t pid = fork();
    exitIf(pid == -1, kForkFail, stderr, "Fork function failed.\n");
    if (pid == 0) exit(110 + i); /* a contrived return value */
}
```
and
```C
for (size_t i = 0; i < N; i++) {
    children[i] = fork();
    if (children[i] == 0) exit(110 + i); /* a contrived return value */
}
```
Once a child process is created, it is **terminated immediately thereafter**, leaving no chance for the child to fork itself. Recall from 2.1.1 that `fork()` produces such a good replica of the original that all information, including the place to which the loop has progressed, are the same. The child process do **not** start from the beginning of the program.

Here is an illustration, with 2 children created. Note that the order of exiting and reaping might not be as same as illustrated.
```
               exit()      (zombie)        reaped
            ┌─────●─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ○        
     child1 │     : return                           
            │     :           exit()      (zombie)     reaped       
            │     :          ┌─────●─ ─ ─ ─ ─ ─ ─ ─ ─ ── ○     
            │     :   child2 │     : return                      
parent      │     ▽          │     ▽                       
────────────┴────────────────┴─────────────────────────────▶
         fork()           fork()
```
###### EOF