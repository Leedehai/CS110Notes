## Topic 2 Multiprocessing (4)
<div id="author-signature">Leedehai</div>
Friday, April 21, 2017<br>Monday, April 24, 2017

## 2.6 Inter-process communication II: signal and signal handler
### 2.6.1 An overview
- *Signal* is a more rudimentary messaging mechanism of inter-process communications. Moreover, it is how process communicates with the OS kernel itself. In C, a signal is represented by a macro-defined constant integer.
> For example: <br>Signal `SIGSEGV` (macro-defined constant `11`) indicates segmentation violation (e.g. writing to an invalid address); `SIGFPE` indicates floating-point exception (e.g. diving by integer `0`); `SIGILL` indicates an illegal instruction; `SIGINT` is forced interruption sent by your Ctrl+C in the shell; `SIGTSTP` and `SIGCONT` can temporarily stop and resume a process. There are also two signals left for the user: `SIGUSR1` and `SIGUSR2`. For a complete list, you can see [here](http://man7.org/linux/man-pages/man7/signal.7.html).

- Upon receiving a signal, the recipient process can either ignore it, abort, or turn to the corresponding signal handler. If a custom signal handler is not registered for the process, the default handler will be invoked.
- After the signal function completes, the process **resumes**, unless the handler terminates the process for good by calling `exit()`.
- For most signals, the system call `signal()` allows you to register a custom signal handler function for it. `signal()`'s prototype is `sighandler_t signal(int sig, sighandler_t handler);`, where `sighandler_t` is of type `void(*)(int)`. To de-register the custom handler, use `SIG_DFL` ("default") as the second argument. Example:
    ```C
    #include <stdlib.h>
    #include <unistd.h>
    #include <stdio.h>
    #include <sys/wait.h>

    static void handleSIGSEGV(int sig) {
        printf("Oops, super sorry, a segfault occurs.\n");
        exit(0); /* Exit the process, otherwise it keeps running into
                    the segfault error each time the process resumes */
    }

    int main() {
        signal(SIGSEGV, handleSIGSEGV); /* register a custom handler */
        *(int *)NULL = 1; /* cause a segmentation fault */

        /* the process can never reach here */
        return 0;
    }
    ```
    If you would like to pass some client data to that signal handler, you can do so by populating a gloabal variable, which is visible to the handler function.

    However, there are some signals to which you cannot register custom signal handlers, such as `SIGCONT` and `SIGKILL`, which can kill a process by force.
> Responding to a signal with signal handler is a form of *exception handling*. That is, the process suspends its normal routine and executes the exeception handling routine, and then goes back to resume its normal routine.
```
                     │             │
     normal routine  │             ▼ instruction sequence
                     │ suspend         
          signal ~~> ▼........▷┐
                     ┌◁...     │
                     │ resume  │ exception handling routine
                     │   :     │
                     │   :.....▼....▷ abort   
                     ▼       
```
Today we will focus on a specific signal: `SIGCHLD` (constant `17`).

### 2.6.2 Dealing with signals: `SIGCHLD` as an example
- A process can use system call `waitpid()` (or `wait()`) to wait for a child's termination and reap the zombie. The problem is, **performing a wait blocks the parent process**.
- Now, we want the parent process to deal with a child's termination (or suspension), without blocking the parent itself. Therefore, we turn to the signal `SIGCHLD` and its signal handler.
- A `SIGCHLD` is sent **automatically** by the kernel to a process, telling the process that one of its child process **terminates** or **stops**.
- The thinking is that: the parent process continues to run until a `SIGCHLD` signal arrives, at which point the parent invokes the signal hander, in which a wait is performed to get the PID and status of the terminated child process, and reap the zombie. Since by the time of wait a child has already terminated, the `waitpid()` can return immediately without blocking the parent process.
```
                                            completes   reaped
            ┌─────────────(do stuff)───────────────●─ ─ ─○ 
     child  │                                      :                 ──▶
            │                              SIGCHLD :                 time
parent      │                                      :  
────────────┴──────────(do other stuff)────────────▽┐ handler ┌───▶
          fork()                                    └─────────┘
```
#### 2.6.2.1 An example of handling `SIGCHLD`
Suppose a parent process creates 5 child processes. The parent goes on to do other stuff (here, sleeping) while waiting for children's termination and reap them, until all five children are reaped.

Since the parent needs to do other stuff while waiting, we cannot use `waitpid()` as it will block the parent process. Here, we need to use `SIGCHLD`.
```C
#include <unistd.h>
#include <stdio.h>
#include <sys/wait.h>
#include "other_stuff.h" // for do_stuff

static int kNumChildren = 5;

static int numDone = 0;

static void reapChild(int sig) { /* the handler. sig is unused. */
  waitpid(-1, NULL, 0); /* wait for any one child and reap it */
  numDone++;
}

int main() {
    signal(SIGCHLD, reapChild); /* register the handler */
    for (int kid = 1; kid <= kNumChildren; kid++) {
        pid_t pid = fork(); /* create a child process */

        if (pid == 0) { /* for the child process */
          sleep(3 * kid); /* sleep for some seconds proportinal to kid */
          printf("Kid %d completes.\n", kid);
          exit(0); /* do not forget to exit! */
        }

        /* for the parent process */
        while (numDone < kNumChildren) {
            printf("Parent wakes up.\n");
            sleep(5);
        }

        printf("All finished. Go home.\n");
        return 0;
    }
}
```
### 2.6.3 Pending signal
A signal that has been sent but not yet received is called a *pending signal*. At
any point in time, there can be **at most one pending signal of a particular type**. If a process has a pending signal of type K, then any subsequent signals of the same type K sent to that process are not queued; they are simply discarded.

If all of the aforementioned 5 child processes (in 2.6.2.1) completes at the same time, the kernel will send 5 `SIGCHLD` signals at once to the parent process, but the signal handler will get executed only once - because the parent process only knows if there is a `SIGCHLD` signal waiting to be handled, but it **has no clue how many such signals have yet to be dealt with**.
> For each process, the kernel maintains the set of pending signals in a **"pending bit" vector** in the kernel memory space. The kernel sets bit K in pending whenever a signal of type K is delivered to that process and clears bit K when this signal is received by that process.

In this hypothetical case, the parent process never completes, as the counter `numDone` never reaches 5. Also, only one children is reaped, because the handler is executed only once.

We have to modify the code like this:
```C
#include <unistd.h>
#include <stdio.h>
#include <sys/wait.h>
#include "other_stuff.h" // for do_stuff

static int kNumChildren = 5;

static int numDone = 0;

static void reapChild(int sig) { /* the handler. sig is unused. */
  while (true) { /* keeps reaping till there is no zombie at the moment */
      /* the 3rd arg should be WNOHANG, otherwise waitpid()
       * blocks the process while waiting if there is
       * at least one child who didn't complete */
      pid_t pid = waitpid(-1, NULL, WNOHANG);

      /* With WNOHANG, if there is >= 1 child running
       * and 0 unreaped zombie child, waitpid() returns 0.
       * If there is no child, waitpid() returns -1 */
      if (pid <= 0) break;
      numDone++;
  }
}

int main() {
    signal(SIGCHLD, reapChild); /* register the handler */
    for (int kid = 1; kid <= kNumChildren; kid++) {
        pid_t pid = fork(); /* create a child process */

        if (pid == 0) { /* for the child process */
          
          sleep(3); /* sleep for the same amount of time */
          printf("Kid %d completes.\n", kid);
          exit(0); /* do not forget to exit! */
        }

        /* for the parent process */
        while (numDone < kNumChildren) {
            printf("Parent wakes up.\n");
            sleep(5);
        }

        printf("All finished. Go home.\n");
        return 0;
    }
}
```
Note that in this modified version we pass `WNOHANG` as the 3rd argument to `waitpid()`.

The reason is this: **if there is still at least one child left running**, `waitpid(-1, NULL, 0)` will block the parent process. Replacing the 3rd argument `0` with `WNOHANG` prevents the parent process from being hanged up in this case, and `waitpid()` will immediately return `0`. 

Of course, if all children have been completed and reaped, `waitpid()` returns `-1` as before, as there is no running child to wait.

Another caveat is that the kernel is responsible for scheduling the parent and child processes. Therefore, the 5 children might not be finished at the same time, even though I told them to sleep for the same amount of time. 
> Each process has a clock to record the time duration of its actual running. When the process is suspended, the clock stops counting.

### 2.6.4 Blocked signal
- A process can selectively *block* the receipt of certain signals. When a signal type is blocked, the signal of this type won't be received by the process until it is unblocked.
- For each process, the kernel also maintains the set of blocked signals in a **"blocked bit" vector**, in the kernel memory space. If a signal is delivered to a process, but that kind of signal is blocked by this process, the process won't receive the signal. In this case, the corresponding bit in the pending signal vector is not cleared until the signal is unblocked and received.
- Blocking certain signals **forces an execution order** of the parent and the child process. Without blocking, the order of process execution is not guaranteed.

#### 2.6.4.1 Blocking/unblocking a signal: `sigprocmask()`
- System call `sigprocmask()` can examine and modify the set of blocked signals.
- Its prototype is
    ```C
    int sigprocmask(int how, const sigset_t *set, sigset_t *oldset);
    ```
    The first argument can be:
    - `SIG_BLOCK`: **adds** the signal types specified by the second argument `set` to the list of already-blocked signal types.
    - `SIG_UNBLOCK`: **removes** the signal types specified by the second argument `set` to the list of already-blocked signal types.
    - `SIG_SETMASK`: **set** to list of blocked signal types to the the ones specified by the second argument.

    The second argument is a pointer of type `sigset_t *`, which points to a bit vector of type `sigset` (it can be viewed as an integer), each bit representing a certain signal type.

    The third argument is also a pointer of type `sigset_t *`. The system call `sigprocmask()` populates the bit vector referenced by this pointer with the **original block list** before alteration. If not interested, it can be `NULL`.
- More about the bit vector `sigset_t`: this is defined in the POSIX extension of C. Typical usages is:
    - Create a `sigset_t` bit vector: `sigset_t mask;` (bit values are uninitialized)
    - Clear this bit vector: `sigemptyset(&mask);`
    - Add a `SIGCHLD` (or another signal type) to this vector: `sigaddset(&mask, SIGCHLD);`

#### 2.6.4.2 An example: blocking `SIGCHLD`
We want a program that executes the `date` command consecutively for 3 times.

Without blocking, the code looks like this (buggy - synchronization problem):
```C
#include ...

static void reapChild(int sig) { /* the SIGCHLD handler */
  pid_t pid;
  while (true) { /* keeps reaping zombies until there is no zombie */
    pid = waitpid(-1, NULL, WNOHANG);
    if (pid <= 0) break;
    printf("Job #(%d) completes and removed\n", pid);
  }
}

int main(int argc, char *argv[]) {
  exitIf(signal(SIGCHLD, reapChild) == SIG_ERR, kSignalFailed,
         stderr, "signal function failed.\n");
  for (size_t i = 0; i < 3; i++) {
    pid_t pid = fork();
    exitIf(pid == -1, kForkFailed,
           stderr, "fork function failed.\n");
    if (pid == 0) {
      char *listArguments[] = {"date", NULL};
      exitIf(execvp(listArguments[0], listArguments) == -1,
             kExecFailed, stderr, "execvp function failed.\n");
    }
    snooze(1); /* represents meaningful time spent */
    printf("Job #(%d) added\n", pid);
  }

  return 0;
}
```
The problem is, the command `date` executes very fast, so the child process usually completes before the parent finishes its `sleep()`, causing the parent process having to turn to the signal handler and print `"Job #(pid) completes and removed"` **before** printing `"Job #(pid) added"` - not good.
```
                     completes   reaped
               ┌──("date")──●─ ─ ─○ 
     child     │            :         
               │            : SIGCHLD
parent         │    0.8s    :           0.2s
───────────────┴────────────▽┐         ┌────●─────────────▶
            fork()           │ handler │  "job added"
                             └───────●─┘
                                "job removed"
```
To fix the synchronization problem, we need to block the `SIGCHLD` signals to **enforce an execution order**. That is, the parent blocks `SIGCHLD` signals until it finishes its sleeping task.

The corrected version is this:
```C
#include ...

static void reap(int sig) {...} /* the SIGCHLD handler */

int main(int argc, char *argv[]) {
  signal(SIGCHLD, reapChild);

  sigset_t mask;
  sigemptyset(&mask);
  sigaddset(&mask, SIGCHLD);

  for (size_t i = 0; i < 3; i++) {
    /* the parent blocks SIGCHLD signals */
    sigprocmask(SIG_BLOCK, &mask, NULL);
    pid_t pid = fork();

    if (pid == 0) { /* for the child process */
      /* The child duplicates the parent's block list,
       * so the child needs to unblock SIGCHLD, in case
       * "date" might rely on this signal */
      sigprocmask(SIG_UNBLOCK, &mask, NULL);

      char *listArguments[] = {"date", NULL};
      execvp(listArguments[0], listArguments);
    }
    /* for the parent process */
    sleep(1);
    printf("Job %d added to job list.\n", pid);
    
    /* the parent unblock SIGCHLD signals,
     * being able to receive SIGCHLD again */
    sigprocmask(SIG_UNBLOCK, &mask, NULL);
  } 

  return 0;
}
```
Note that `execvp()` does not change a process's blocked signal list, as the list is maintained by the kernel, not residing in the process's memory space.
```
                         completes                        reaped
               ┌──("date")──●─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ○ 
     child     │            : SIGCHLD      
               │            :. .. .. .. .. .. .. .. ...       
parent       ▬▬│▬▬▬▬▬b▬▬l▬▬o▬▬c▬▬k▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬:  
────────────◉──┴─────────────────────────────●───────◉▽┐         ┌───▶
        block  fork()        1.0s   "job added" unblock│ handler │
      SIGCHLD                                   SIGCHLD└───────●─┘
                                                          "job removed"
```
> SIDE NOTE:<br>A *job* refers to a family of processes that is (directly or indirectly) initiated by a command in shell. You can print all current running jobs and their job ID using command `jobs`. It is a concept which is only meaningful for the shell, not for the OS kernel.

### 2.6.5 Send a signal: `kill()`
- The system call `kill()` can be used to send any signal to any process.
- Its prototype is `int kill(pid_t pid, int sig);`.
- The signal `sig` is sent to the process identified by `pid`. If `pid` is less than `-1`, then `sig` is sent to every process belonging to the process group whose pgid is the absolute value of `pid`.
- This system call is a **misnomer** - it can deliver signals more than `SIGKILL`.
- In shell, there is a command also named `kill`, which can send a signal (`SIGKILL` by default) to the process whose PID is provided as the command's argument. If you would like to send the process 1009 with a signal other than `SIGKILL`, say, `SIGSEGV` (whose value is `11`), you could use the command like this: `$ kill -11 1009`, in which case, process 1009 would be delivered a signal claiming there is a segmentation fault.

#### 2.6.5.1 The kill puzzle
Consider the following program. What is the output?
```C
#include ...

static pid_t pid;
static int counter = 0;

static void parentHandler(int unused) {
  counter++;
  printf("counter = %d\n", counter);

  kill(pid, SIGUSR1); /* the parent sends the child a signal */
}

static void childHandler(int unused) {
  counter += 3;
  printf("counter = %d\n", counter);
}

int main(int argc, char *argv[]) {
  signal(SIGUSR1, parentHandler); /* the parent register its handler */
  if ((pid = fork()) == 0) { /* for the child */
    signal(SIGUSR1, childHandler); /* the child re-register the handler */

    kill(getppid(), SIGUSR1); /* the child sends the parent a signal */
    return 0;
  }
  
  /* for the parent */
  waitpid(pid, NULL, 0);
  counter += 7;
  printf("counter = %d\n", counter);
  return 0;
}
```
The output depends on the kernel's scheduler. Because the child process may or may not exit before `parentHandler` has a chance to signal it.
```
$ ./kill-puzzle
counter = 1
counter = 8
$ ./kill-puzzle
counter = 1
counter = 3
counter = 8
$
```
Illustrations might be helpful.
```
            re-register                             Case 1-1:
            handler     completes  reaped           
            ┌───◉─────●──●─ ─ ─ ─ ─ - ○             The child exists
     child  │ c=0     :                             before the parent
            │         : SIGUSR1                     signals it.
parent  c=0 │         :        waitpid()            (Version 1)
──────◉─────┴─────────▽┐         ┌────●────●──▶
register  fork()       └──●──────┘       (1+7)"8"   print: 1 8
handler             (0+1)"1"  c=1           c=8
```
```
            re-register                             Case 1-2:
            handler         completes            
            ┌───◉─────────●─●─ ─ - ○ reaped         The child exists
     child  │ c=0         :                         before the parent
            │             : SIGUSR1                 signals it.
parent  c=0 │  waitpid()  :  resume:waitpid()       (Version 2)
──────◉─────┴─────●───────▽┐      ┌●────●─────▶
register   fork()          └──●───┘   (1+7)"8"      print: 1 8
handler                 (0+1)"1" c=1     c=8
```
```
            re-register   (0+3)"3"  c=3              Case 2-1:
            handler          ┌──●─────┐completes             
            ┌───◉─────●─────△┘        └─●○ reaped    The child didn't
     child  │ c=0     :     :                        exit before the
            │  SIGUSR1:     :SIGUSR1                 parent signals it.
parent  c=0 │         :     :   waitpid()            (Version 1)
──────◉─────┴─────────▽┐    :    ┌────●────●──▶
register  fork()       └──●─●────┘       (1+7)"8"    print: 1 3 8
handler             (0+1)"1"   c=1          c=8
```
```
            re-register   (0+3)"3" c=3               Case 2-2:
            handler          ┌──●──┐completes             
            ┌───◉─────●─────△┘     └─●- ○ reaped     The child didn't
     child  │ c=0     :     :                        exit before the
            │  SIGUSR1:     :SIGUSR1                 parent signals it.
parent  c=0 │         :     :   waitpid()            (Version 2)
──────◉─────┴─────────▽┐    :    ┌──────●──●──▶
register  fork()       └──●─●────┘       (1+7)"8"    print: 1 3 8
handler             (0+1)"1"  c=1           c=8
```
### 2.6.6 Send a signal to itself: `raise()`
- The system call `raise()` can send a signal to the calling process itself, taking the signal value as its sole argument.
- A typical use is `raise(SIGKILL)`, which makes a process to commit suicide. For example, you can allow a process to kill itself if it encounters an error.

### 2.6.7 Temporarily lift signal block and suspend for signal: `sigsuspend()`
- The system call `sigsuspend()` can temporarily override signal block **and suspend the calling process** to wait for the intended signal(s) specified in its argument (of type `sigset_t`).
- Once the intended signal arrives, the calling process turns to the corresponding signal handler.
- Once the signal handler returns, `sigsuspend()` will return, and the signal block, if applied before the call to `sigsuspend()`, will bee restored.

An example:
```C
sigset_t additionalMask, exisitngMask;
sigemptyset(&aditionalMask);
sigaddset(&additionalMask, SIGCHLD);

/* Block SIGCHLD */
sigprocmask(SIG_BLOCK, &additionalMask, &existingMask);

/* your code that needs to be executed
 * before the SIGCHLD's handler goes here... */

while (hasRunningChild()) {
  /* Temporarily lift the block, suspend the
   * calling process and wait for SIGCHLD */
  sigsuspend(&existingMask);
  /* Upon return from the signal handler, 
   * sigsuspend() also returns, and the
   * original the signal block is restored */
}

/* Unlock SIGCHLD */
sigprocmask(SIG_UNBLOCK, &additionalMask, NULL);
```
###### EOF