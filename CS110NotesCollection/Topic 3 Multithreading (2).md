# Topic 3 Multithreading (2)
<div id="author-signature">Leedehai</div>
Monday, May 1, 2017<br>Firday, May 5, 2017

## 3.3 C++ form of multithreading: `<thread>`
- The C++ thread interfaces are type-safe and cleaner.

### 3.3.1 The old-school C form:
```C
/* C */
#include <stdio.h>
#include <pthread.h>

/* thread routine */
/* Note the return and arg types are all "void *" */
void *recharge(void *unused) {
    printf("I recharge by spending time alone.\n");
    return NULL;
}

int main() {
    pthread_t introverts[6];
    for (size_t i = 0; i < 6; i++) {
        pthread_create(&introverts[i], NULL, recharge, NULL);
    }
    for (size_t i = 0; i < 6; i++) {
        pthread_join(introverts[i], NULL);
    }

    return 0;
}
```
### 3.3.2 The C++ form:
> `using namespace std;` is omitted in C++ code afterwards.
```C++
/* C++11 */
#include <iostream>
#include <thread>
/* CS110 iomanipulators (oslock, osunlock) used to lock down streams */
#include "ostreamlock.h"
using namespace std;

/* Note the return type is "void" */
void recharge() {
    cout << oslock 
         << "I recharge by spending time alone." << endl
         << osunlock;
    return;
}

int main() {
    /* declare 6 thread objects (handles) */
    thread introverts[6];
    
    /* install thread routine to these threads */
    for (thread &t : introverts) {
        t = thread(recharge);
    }
    /* join: block, wait, reap zombie threads */
    /* note you need the "&" below, since thread class's
     * assignment operator is deleted */
    for (thread &t : introverts) {
        t.join();
    }

    return 0;
}
```
C++ prototypes explained:
- `oslock` and `osunlock` are implemented by CS110 course and meant to exclusively lock/unlock the output file stream, since `cout` is not thread-safe (unlike `printf()`). Without them, `ostream::operator<<()`'s output by multiple threads may get garbled.
- In C++, the thread routine can **take in any number of arguments** of any type.
- In C++, the thread routine **returns nothing**. If it does return something, the return value will be ignored.
- `thread` is a C++11-provided class, defined in the `std` namespace.
    - These methods are used in the code above:
        ```C++
        /* default constructor */
        thread();
        /* initialization (the exact prototype is not our focus):
           takes in a function pointer, followed by arguments (if any)
           the thread routine's return is ignored */
        thread (pRoutine, args...);

        /* assign operator */
        thread& operator= (thread&& rhs); // rvalue reference
        /* join */
        void join();
        ```
    - Some other methods that may be frequently used:
        ```C++
        /* check is joinable */
        bool joinable();
        /* detach */
        void detach();
        /* get thread ID */
        thread::id get_id();
        ```
> SIDE NOTE:<br>The method `thread &operator=(thread &&t)` takes in an rvalue reference. A effect of this design is that, after the `thread` object `t` on right hand side is assigned to the left hand side, **the original** `t` **is gutted**: as if it were a zero-argument constructed, empty, **detached** thread object.<br>Such mechanism is suitable if the right hand size object takes considerable effort to construct or clone.<br>It is unlike normal `a = b` assignment, after which `b` is intact and `a` is a copy of `b`.

### 3.3.3 Another C++ example: using variadic arguments
```C++
/* C++11 */
#include ...

/* Note the return type is "void" and we pass in an arg to it*/
void greet(size_t id) {
  for (size_t i = 0; i < id; i++) {
    cout << oslock << "Greeter #" << id << " says 'Hello!'" << endl << osunlock;
  }    
  cout << oslock 
       << "Greeter #" << id << " has issued all of his hellos, "
       << "so he goes home!" << endl 
       << osunlock;
}

int main(int argc, char *argv[]) {
  thread greeters[6];
  for (size_t i = 0; i < 6; i++) {
      /* Note that the variadic arg is passed in (one arg here):
         thread (pRoutine, args...); */
      greeters[i] = thread(greet, i + 1);
      /* Pay attention to avoid the race condition
         as discussed in 3.2.1 */
  }
  for (thread& greeter: greeters) {
      greeter.join();
  }

  return 0;
}
```

## 3.4 Race condition: critical region, `mutex` (KOB)
- Consider there are two ticket agents at the [United Airlines](https://en.wikipedia.org/wiki/United_Express_Flight_3411_incident). Their job is to sell 3 tickets in total, and the law prohibits overselling.
### 3.4.1 Buggy ticket selling:
```C++
/* Buggy */
#include <iostream>
#include <thread>
#include "ostreamlock.h"
using namespace std;
static int numTickets = 3; /* threads cannot share automatic variables */

 7 void ticketAgent(size_t id) {
 8     while (numTickets > 0) {
 9         numTickets--;
10         cout << oslock 
11              << "Ticket agent " << id << " sold one ticket,"
12              << " there are " << numTickets 
13              << " more to sell!" << endl
14              << osunlock;
15     }
16}

int main() {
    thread agents[2];
    for (size_t i = 0; i < 2; i++) {
        agents[i] = thread(ticketAgent, 101 + i); /* agent 101, 102 */
    }
    for (thread &a : agents) { /* block and wait for each thread */
        a.join();
    }
    cout << "End of business day!" << endl;

    return 0;
}
```
There are two problems caused by the **unpredictability of the scheduler**.
> Pause and think: which two?
- The first problem.
    - Say, (1) agent 101 proceeds to line 8 and discovered that `numTickets` is `1`, and is thus indeed greater than `0`, so it decides to sell one more ticket. (2) immediately after line 8, agent 101's thread is switched out from CPU, and agent 102 comes in. (3) agent 102 comes along and proceeds to line 8, it discovers `numTickets` is `1` and is greater than `0`, so it proceeds to sell the ticket and decrement `numTickets` to `0`. (3) agent 101 is switched back and it proceeds from where it left - it sells one ticket, decrementing `numTickets`.
    - In the scenario above, the final `numTickets` is `-1`, and one ticket is oversold.
    - The root of this problem is that, the code lacks a mechanism to guarantee line 9 is executed immediately after line 8.
- The second problem.
    - Let's assume the first problem is fixed: line 9 is executed immediately after line 8.
    - Note that line 9: `numTickets--;` might be translated into at least 3 assembly instructions, not 1 (assume `numTickets` is stored on address `0x1122`) - dependent on the instruction set:
        ```
        MOVQ 0x1122, %rax   ;move value from 0x1122 to RAX  (line A)
        SUB %rax, $1        ;subtract the value in RAX by 1 (line B)
        MOVQ %rax, 0x1122   ;move value from RAX to 0x1122  (line C)
        ```
    - Say (1) agent 101 is switched out from CPU immediately after it completes line A, and the RAX value is stored. (2) agent 102 comes along, sells a ticket, and decrements `numTickets` to `0`. (3) agent 101 is switched back, stored register values loaded back to CPU, and proceeds from where it left: it executes line B and C.
    - Thus, the final `numTickets` is `-1`, and one ticket is oversold. **A C++ statement is not atomic; an assembly instruction is**.
    ```
    <thrd.1> :                      : <thrd.1>
    ───●───●─:─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─●──●─..─●─▶
       {   A :                      :B  C    }  
             ▽                      △             ──▶ time
             ▽                      △             A: MOVQ 0x1122, %rax
             :  {    A  B  C      } :             B: SUB %rax, $1
    ─ ─ ─ ─ ─:──●────●──●──●─..───●─:─ ─ ─ ─ ──▶  C: MOVQ %rax, 0x1122
             :      <thrd.2>        :
    ```

> A *race condition* happens when the processes or threads depend on some **shared state** (`numTickets`'s value here). <br>Operations upon shared states are critical code sections (*critical region*s) that must be mutually exclusive. That is, when one steps into the critical region, the others should be prevented to do the same until the first one steps out.

### 3.4.2 Solution: `mutex` and locking
- In the example above, C++ code line 8 to 14 should be executed in tandem without the thread being switched halfway. That is, line 8 to 14 should be a transaction.
- "mutex": mutual exlusion. This is a C++ class, and some of its methods are
    ```C++
    /* constructor */      mutex();
    /* lock */             void lock();
    /* unlock */           void unlock();
    /* lock if unlocked */ bool try_lock();
    ```
    - `lock()`: (1) if the object is unlocked, then lock it, and the following code is in a transaction; (2) if the object is locked by another thread, then **block and wait** until unlocked; (3) if the object is already locked by the same thread, it caused an undefined behavior.
        > Whenever a process or thread is in a blocked state, it will be switched out from CPU, and register values are stored in memory.
        > When more than one thread attempts to lock it, one of them succeeds.

    - `unlock()`: (1) if the object is locked by the same thread, then unlock it, and the following code is not in a transaction; (2) if the object is already unlocked, the behavior is undefined.

    - `try_lock()`: (1) if the object is unlocked, then lock it and return `true`; (2) if the object is locked by another thread, then give up and immediately return `false`; (3) if the object is already locked by the same thread, it caused an undefined behavior.

    > Like in a hostel, people go to the bathroom to take shower. The shower-taker locks the door before start and unlocks it after the job is done. While the door is locked, other people just wait outside (or leave and come back later).

The fixed version:
```C++
/* Problem fixed */
#include <iostream>
#include <thread>
#include "ostreamlock.h"
#include <mutex>        /* for mutex class */
using namespace std;
static int numTickets = 3;
static mutex ticketLock;   /* global, shared among threads */

 7 void ticketAgent(size_t id) {
 8*   while (true) {
 8+       ticketLock.lock(); /* lock */
 9        numTickets--;
10        cout << oslock 
11             << "Ticket agent " << id << " sold one ticket,"
12             << " there are " << numTickets 
13             << " more to sell!" << endl
14             << osunlock;
14+       if (numTickets == 0) break;
14++      ticketLock.unlock(); /* unlock */
15    }
16}

int main() {
    thread agents[2];
    for (size_t i = 0; i < 2; i++) {
        agents[i] = thread(ticketAgent, 101 + i); /* agent 101, 102 */
    }
    for (thread &a : agents) { /* block and wait for each thread */
        a.join();
    }
    cout << "End of business day!" << endl;

    return 0;
}
```
Code explained:
- To fix problem 1: replace `while(numTickets < 0)` with `while (true)` and place the condition checking inside the loop. And...
- To fix problem 1 and 2: use a `mutex` lock object to guard the loop body, which is the *critical region*: the sequence of instructions that **at most one thread** should be in at any time.
- Note that the `mutex` object should be global, so all threads can agree on this lock.
    > You certainly don't want someone else to come into the bathroom **though another door** while you are in it, right?

## Appendix: the implementation of `oslock` and `osunlock`
```C++
/* ostreamlock.h */
#include <ostream>

std::ostream& oslock(std::ostream& os);
std::ostream& osunlock(std::ostream& os);
```
```C++
/* ostreamlock.cc */
#include <ostream>
#include <iostream>
#include <mutex>
#include <memory>
#include <map>
#include "ostreamlock.h"
using namespace std;

static mutex mapLock;
static map<ostream *, unique_ptr<mutex>> streamLocks;

ostream& oslock(ostream& os) {
  ostream *ostreamToLock = &os;
  if (ostreamToLock == &cerr) ostreamToLock = &cout;
  mapLock.lock();
  unique_ptr<mutex>& up = streamLocks[ostreamToLock];
  if (up == nullptr) {
    up.reset(new mutex);
  }
  mapLock.unlock();
  up->lock();
  return os;
}

ostream& osunlock(ostream& os) {
  ostream *ostreamToLock = &os;
  if (ostreamToLock == &cerr) ostreamToLock = &cout;
  mapLock.lock();
  auto found = streamLocks.find(ostreamToLock);
  mapLock.unlock();
  if (found == streamLocks.end())
    throw "unlock inserted into stream that has never been locked.";
  unique_ptr<mutex>& up = found->second;
  up->unlock();
  return os;
}
```
###### EOF