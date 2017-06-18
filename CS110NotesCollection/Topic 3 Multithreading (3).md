# Topic 3 Multithreading (3)
<div id="author-signature">Leedehai</div>
Firday, May 5, 2017<br>Monday, May 8, 2017

> When coding with threads, you need to ensure: <br>- that there are no possibility of **race condition**s, and...<br>- that there's no possibility of **deadlock**.

## 3.5 The threat of deadlock: dining philosophers problem (KOB)
`mutex` is not the panacea of all synchronization problems in multithreading. In fact, `mutex` can solve the problem of race conditions (as in 3.4), but not the one we are going to describe below.
> In computer science, the [dining philosophers problem](https://en.wikipedia.org/wiki/Dining_philosophers_problem) is an example problem often used in concurrent algorithm design to illustrate synchronization issues and techniques for resolving them. It was originally formulated in 1965 by [Edsger Dijkstra](https://en.wikipedia.org/wiki/Edsger_W._Dijkstra) as an exam problem.
```
                    .─.                THE SETUP:   
                   (   ) Plato            - A circular table,
                    `─'                   - N philosophers (N >= 2), 
       ┌──────────────────────────┐       - N forks, each placed between
       │  \       ┌────┐       /  │         two adjacent philosophers,
       │   \      │bowl│      /   │       - M meals for each philosopher,
       │          └────┘          │       - Each philosopher needs 2     
       │       ┌──────────┐       │           adjacent forks to eat
 .─.   │┌────┐ │          │ ┌────┐│   .─.     spaghetti (yeah, weirdos),
(   )  ││bowl│ │spaghetti │ │bowl││  (   )    otherwise he philosophizes.
 `─'   │└────┘ │          │ └────┘│   `─'     
Hegel  │       └──────────┘       │ Descartes  
       │          ┌────┐          │            
       │   /      │bowl│      \   │     THE PROBLEM:  
       │  /       └────┘       \  │       How should each philosopher
       └──────────────────────────┘       behave so no one starves  
                    .─.                   i.e. each one is able to
 N = 4 here, but   (   ) Rousseau         eat M times?
 We use N = 5 below `─'                                    
```
- Note that we will assume `N` = 5 and `M` = 3. That is, 5 philosophers, 5 forks, 3 meals for each.
### 3.5.1 Version 1: deadlock happens with nonzero probability
- The most naïve approach is simply having N forks represented by N locks, and having each philosopher lock the fork before using and unlock it afterwards. There is no race for the fork here, since each fork has a lock on it, but there is another problem.

Let's look at the code first.
```C++
/* Verion 1: deadlock may happen */
#include ...
using namespace std;

static const unsigned int kNumPhilosophers = 5;
static const unsigned int kNumForks = kNumPhilosophers;
static const unsigned int kNumMeals = 3;

/* forks modeled as mutexes */
static mutex forks[kNumForks];

static void think(unsigned int id) {
  cout << oslock << id << " starts thinking." << endl << osunlock;
  sleep_for(getThinkTime()/* random number, not important here */);
  cout << oslock << id << " all done thinking. " << endl << osunlock;
}

static void eat(unsigned int id) {
  unsigned int left = id;
  unsigned int right = (id + 1) % kNumForks;

  forks[left].lock();    /* lock, if already locked, then wait */
  forks[right].lock();   /* lock, if already locked, then wait */

  cout << oslock << id << " starts eating." << endl << osunlock;
  sleep_for(getEatTime());
  cout << oslock << id << " done eating." << endl << osunlock;

  forks[left].unlock();  /* unlock */
  forks[right].unlock(); /* unlock */
}

static void philosopher(unsigned int id) {
  for (unsigned int i = 0; i < kNumMeals; i++) {
    think(id);
    eat(id);
  }
}

int main(int argc, const char *argv[]) {
  thread philosophers[kNumPhilosophers];
  for (unsigned int i = 0; i < kNumPhilosophers; i++) 
    philosophers[i] = thread(philosopher, i);
  for (thread& p: philosophers)
    p.join();

  return 0;
}
```
- What's the problem?
    - **each** philosopher emerges from his deep thinking, successfully grabs the fork to his left, and then gets pulled off the processor because his time slice is over.
    - If this pathological scheduling pattern presents itself, eventually all each philosopher will be not able to grab the fork on his right, as that fork is already locked by the philosopher on his right.
    - That will leave the program in a state where all threads are entrenched in a state of *deadlock*, because each philosopher is stuck waiting for the philosopher on his right to let go of his fork.
    - The probability of this happening increases there is a sleep between locking the left fork and the right, as a thread switch will most likely happen during this sleep period.
> A *deadlock* is a situation in which two or more competing actions are each waiting for the other to finish, and thus neither ever does.<br>If a race condition happens, the program can still proceed (though potentially with erroneous results), but if a deadlock happens, the program falls into a never-ending wait, like an obscure while-true loop.
- The solution, however, might not be intuitive. Solutions are many; one heuristic, to prevent the deadlock from happening, is to introduce the notion of *permission slip*s (这就是“条子”，哈哈哈), or *permit*s.

### 3.5.2 Version 2: permission slips, limit the number of bidding threads
- Deadlock can be programmatically prevented by implanting directives to limit the number of threads that try to participate in an action that could otherwise result in deadlock.
    - (1) We could, for instance, recognize that it's impossible for 3 or more philosophers to be eating at the same time, via a simple pigeonhole principle argument (e.g. 3 philosophers can be eating at the same time if and only if there are 6 forks, and there are not). We can, therefore, limit the number of philosophers grabbing forks to 2.
        > Multithreading purists may criticize that this approach, by limiting the number of participating threads from 5 to 2, sacrifices the power of multithreading too much.

    - (2) We can also argue that it's okay to let up to 4 (but not all 5) philosophers to transition into the `eat()` function of their think-eat cycle, knowing that at least one will succeed in grabbing both forks. That is, we allow up to 4 philosophers to take part in the bid for forks at the same time.
        > We will take this approach, as this approach downsize the number of threads that are allowed to grab fork from 5 to 4, not from 5 to 2, so it lets all threads to make as much progress as possible.
```C++
/* Version 2: no bug, but has busy-waiting */
#include ...
using namespace std;

static const unsigned int kNumPhilosophers = 5;
static const unsigned int kNumForks = kNumPhilosophers;
static const unsigned int kNumMeals = 3;

/* forks modeled as mutexes -- to solve the race condition */
static mutex forks[kNumForks]; 

/* impose limit on # bidding threads -- to solve the deadlock */
static unsigned int numAllowed = kNumPhilosophers - 1;
/* the lock partnered with numAllowd */
static mutex numAllowedLock;

static void think(unsigned int id) {
  /* same as in 3.5.1 */
}

/* wait to get a permission slip to participate in the bid for forks */
static void waitForPermission() {
  while (true) {
    numAllowedLock.lock();
    if (numAllowed > 0) break;
    numAllowedLock.unlock();
    sleep_for(10);
  }
  numAllowed--;
  numAllowedLock.unlock();
}

/* give back the permission slip to the pool so others can participate */
static void grantPermission() {
  numAllowedLock.lock();
  numAllowed++;
  numAllowedLock.unlock();
}

static void eat(unsigned int id) {
  unsigned int left = id;
  unsigned int right = (id + 1) % kNumForks;

  waitForPermission(); /* wait for permission until get one */

  forks[left].lock();  /* may wait here */
  forks[right].lock(); /* may wait here */

  cout << oslock << id << " starts eating." << endl << osunlock;
  sleep_for(getEatTime());
  cout << oslock << id << " all done eating." << endl << osunlock;

  grantPermission(); /* give back the permission to the pool */

  forks[left].unlock();
  forks[right].unlock();
}

static void philosopher(unsigned int id) {
  /* same as in 3.5.1 */
}

int main(int argc, const char *argv[]) {
  /* same as in 3.5.1 */
}   
```
- It solves the problem of deadlock. It does, however, have one design flaw: the solution uses busy waiting in `waitForPermission()`, which is usually a big no-no. In this function,
    - The philosopher thread continually poll the value of `numAllowed` until the value is positive. At that point, decrement it to emulate the consumption of a shared resource — in this case, a permission slip allowing a philosopher to start trying to grab forks.
    - Since there are multiple threads potentially examining and decrementing `numAllowed`, identify the critical region as one that needs to be guarded by a `mutex`. And if the philosopher notices the value of `numAllowed` is 0 (i.e. all permissions slips are out), then release the lock and yield the processor to some other philosopher thread who can actually do some useful work.
    - The above solution uses *busy waiting*, which is a concurrency jargon used when a thread periodically checks to see whether some condition has changed so it can move on to do more meaningful work.
    - The problem with busy waiting, in most situations, is that the busy-waiting thread **occupies the CPU during its time windows, wasting the CPU time**, which would be better spent to ensure other threads — who presumably have meaningful work — to proceed.
- A better solution: if a philosopher doesn't have permission to advance (i.e. `numAllowed` is confirmed to be zero), then that thread should be **put to sleep indefinitely** until some other thread sees a reason to **wake it up**. In this example, another philosopher thread, after it increments `numAllowed` within `grantPermission()`, could notify the indefinitely blocked thread that some permissions slips are now available.

### 3.5.3 Version 3: improve the solution - no busy-waiting
- We can get rid of the busy-waiting situation by putting the thread into **sleep** and having another thread **wake it up** when some condition is met.
- Implementing this idea requires a more sophisticated concurrency directive that supports a different form of thread communication. Fortunately, C++11 provides a standard, albeit difficult-to-understand, class called the `condition_variable_any`.

```C++
/* Version 3: no bug, no busy-waiting */
#include ...
using namespace std;

static const unsigned int kNumPhilosophers = 5;
static const unsigned int kNumForks = kNumPhilosophers;
static const unsigned int kNumMeals = 3;

/* forks modeled as mutexes -- to solve the race condition */
static mutex forks[kNumForks];

/* impose limit on # bidding threads -- to solve the deadlock */
static unsigned int numAllowed = kNumPhilosophers - 1;
/* the lock meant to protect numAllowed against ++, -- */
static mutex numAllowedLock;

/* to solve the busy-wait introduced in 3.5.2 */
static condition_variable_any cv;

static void think(unsigned int id) {
  /* same as in 3.5.1 */
}

/* wait to get a permission slip to participate in the bid for forks */
static void waitForPermission() {
  lock_guard<mutex> lg(numAllowedLock);

  /* test: if condition is met: proceed
           otherwise: release lock, sleep & wait
     notified: re-acquire lock, re-test
   */
  cv.wait(numAllowedLock, []{ return numAllowed > 0; });

  numAllowed--;
}

/* give back the permission slip to the pool so others can participate */
static void grantPermission() {
  lock_guard<mutex> lg(numAllowedLock);
  numAllowed++;
  if (numAllowed == 1) { /* if numAllowed goes from 0 to 1 */
      /* notify cv to re-test the sleeping threads' condition */
      cv.notify_all();
  }
}

static void eat(unsigned int id) {
  /* same as in 3.5.2 */
}

static void philosopher(unsigned int id) {
  /* same as in 3.5.1 and 3.5.2 */
  /* 1. wait for permission until get one,
     2. lock both forks (may wait here),
     3. enjoy the spaghetti,
     4. give back the permission to the pool,
     5. unlock both forks */
}

int main(int argc, const char *argv[]) {
  /* same as in 3.5.1 and 3.5.2 */
} 
```
- The `condition_variable_any` (abbreviated as "cva" below) is the core concurrency directive that can preempt and block a thread until some condition is met.
- In this example, the philosopher seeking permission to eat waits indefinitely until some condition is met (unless the condition is met already, in which case it doesn't need to wait), by calling `cva::wait()`.
    - If `numAllowed` is positive at the moment wait is called, then it returns immediately without blocking.
    - If `numAllowed` is zero at the moment `cva::wait()` is called, then the calling thread is **pulled off the CPU, marked as blocked** until the thread manager is informed (by another thread) that the value of `numAllowed` has changed.
- In this example, the philosopher just finishing up a meal increments `numAllowed`, and if the value of `numAllowed` goes from 0 to 1, the same philosopher signals (via `cva::notify_all()`) all threads blocked by `cva` that a meaningful update has occurred. That prompts the thread manager to reexamine the condition on behalf of all threads blocked by `cva::wait()`, and potentially allow one or more of them to emerge from their long nap and move on to the work they weren't allowed to move on to before.
- Because `numAllowed` is being examined and potentially changed concurrently by many threads, and because a condition framed in terms of it (that is, `numAllowed > 0`) influences whether a thread blocks or continues uninterrupted, we still need a traditional `mutex` here so that competing threads can lock down **exclusive access** to `numAllowed`.
- Before `cva::wait()` is called, the the supplied `mutex` lock should have been lock already. If `cva::wait()` notices that the supplied condition isn't met, the `cva` object puts the current thread to **sleep** indefinitely and **automatically release**s the lock. When the `cva` object is notified and a waiting thread is switched back on CPU, it **automatically re-acquire**s the `mutex` lock which it released just prior to sleeping, and **re-evaluate the condition** for that thread. If the condition is met, then the thread proceeds to the following lines; otherwise, it automatically releases the lock again and put the thread back in sleep, and then waits for another notification.
    > This is KOB.
```                                       
First call (lock should be locked already)
          │
          ▼          ┌───────────────┐
          ├◀─success─┤re-acquire lock◀────┐  sleep: blocked (off CPU)
          │          └──┬───────▲────┘    │         & wait for an event
 ┌────────▼───────┐    fail    lock       │         of interest
 │   condition?   │     │    available    │
 └────────┬───────┘   ┌─▼───────┴─┐       │
          ├──false──┐ │   sleep   │       │
          │         │ └───────────┘       │
          │  ┌──────▼─────┐               │
        true │release lock│               │
          │  └──────┬─────┘         notification <= notify_all()
          │  ┌──────▼─────┐               │
          │  │   sleep    │───────────────┘
          │  └────────────┘
┌─────────▼─────────┐
│ continue to hold  │
│  lock & proceed   │     FLOWCHART:
└─────────┬─────────┘     void wait(Lock &lock, Predicate condition); 
          ▼                                     
```
- `codition_variable_any` class in summary:
    ```C++
    /* constructor */
    condition_variable_any();

    /* wait: blocks the current thread 
       until the condition variable is woken up */
    template<typename Lock, typename Predicate> 
    void wait(Lock &lock, Predicate condition);

    /* notify all threads blocked by cva */
    void notify_all();

    /* other methods... */
    ```
    - Predicate: a function that returns a boolean. You can pass in a function pointer to a predicate, or pass in a [lambda expression](https://msdn.microsoft.com/en-us/library/dd293608.aspx) that is a predicate. If the predicate returns `false`, then `cva::wait()` puts the calling thread into sleep.
    > You should avoid using the one-argument version of `wait()`, which takes the lock as the only argument, because this version of `wait()` sometimes returns without being notified - "[spurious wake](https://www.justsoftwaresolutions.co.uk/threading/condition-variable-spurious-wakes.html)".
- The `lock_guard` class exists to **automate the locking and unlocking of a** `mutex`.
    - The `lock_guard()` constructor binds an internal reference to the supplied `mutex` object and calls `lock()` on it (if the `mutex` lock is already locked by another thread, the constructor is blocked inside, until the lock is released).
    - The `~lock_guard()` destructor releases the lock on the same `mutex` object.
    - The overall effect: the code section from the constructor being called to the destructor being called (often implicitly at function return or exception throwing) is marked as a `mutex`-protected critical region.
    ```C++
    {                          {
      ...                        ...
      m.lock();                  lock_guard<mutex> lg(m); /* lock m */
      do_something();   <=>      do_something();
      m.unlock();              } /* the destructor unlocks m */
    }                                  
    ```
    > SIDE NOTE:<br>Java: `lock_guard` is C++ STL's answer to Java keyword `synchronized`.

    > SIDE NOTE:<br>The `lock_guard` is a template class, because other than `mutex` class, some other classes like `recursive_mutex` and `timed_mutex` have such `lock()` and `unlock()` methods, too. 
    
    Its prototype is

    ```C++
    /* template declaration */
    template<typename BasicLockable> lass lock_guard;

    /* constructor (the basic one) */
    explicit lock_guard(BasicLockable &m);

    /* destructor */
    ~lock_guard();
    ```
- A pedendic note: to let the other threads to proceed as much as they can, in the `eat()` function, you can place `grantPermission();` immediately after successfully locking the left and right forks, since now that the thread succeeded in locking the two forks already, there is no reason to continue to hold the permission slip - it can give the slip back to the pool, knowing that another thread may grab the slip and try to lock forks.
    > Placing `grantPermission();` after unlocking the forks is acceptable as well, but it has the opposite effect - it further delays the progress of other threads.
- Another pedendic note: the global variable `numAllowed`, i.e. the number of available permission slips, keeps track of the bidding situation of a resource that is at premium - like the forks, which is fewer than the dining philosopher threads, or network connections, or database access, ...
###### EOF