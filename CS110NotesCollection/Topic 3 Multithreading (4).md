# Topic 3 Multithreading (4)
<div id="author-signature">Leedehai</div>
Monday, May 8, 2017<br>Monday, May 15, 2017

## 3.6 The semaphore class
- `semaphore` is implemented by the instructor, a class that is meant to be a high-level abstraction of this trio in 3.5.3:
    ```C++
    /* permission slips to bid for some resource instances */  
    size_t numAllowed;
    /* mutex lock to protect permission slip ++, -- */
    mutex m;
    /* condition variable: used to block and wait
       until a permission slip is available */
    condition_variable_any cv;
    ```
    Similar class is provided in POSIX C extension, Java, Python, .. yet missing from C++ STL.
- `semaphore` API:
    ```C++
    /* semaphore.h */
    #include ...

    class semaphore {
    public:
      semaphore(int value = 0) : value(value) {}
      void signal(); /* value++ */ 
      void wait();   /* value--, potentially blocking */
    private:
      int value;  /* the number of available resource instances */
      std::mutex m;
      std::condition_variable_any cv;

      semaphore(const semaphore& orig) = delete;
      const semaphore& operator=(const semaphore& rhs) const = delete;
    };
    ```
    Design decsions:
    - The instructor designed the API such that there's no getValue-like method. Reason: in between the time you call it and act on it, some other thread could very well change it. Concurrency directives themselves shouldn't encourage anything that might lead to a race condition or a deadlock. Providing this method might encourage you to make this mistake.

    - The instructor removed the copy constructor and assignment operator (using the `delete` keyword), because neither the `mutex` nor `condition_variable_any` object is copy constructable, copy assignable, or even movable.
- `semaphore` implementation:
    ```C++
    #include ...

    void semaphore::signal() {
        std::lock_guard<mutex> lg(m); /* m.lock() to protect value */
        value++;

        /* notify cv that value becomes > 0 */
        if (value == 1) cv.notify_all();
    } /* lock_guard, as a local, is destructed here and m is unlocked */

    void semaphore::wait() {
        std::lock_guard<metex> lg(m); /* m.lock() to protect value */

        /* make sure value > 0 before proceeding,
           if not, block & wait here */
        cv.wait(m, [this]{return value > 0;});

        value--;
    } /* lock_guard, as a local, is destructed here and m is unlocked */
    ```
    - There is a `this` pointer in the closure of the lambda expression, because the lambda function needs the access to the `value` member, which is neither a global variable nor a local one inside the lambda function's body. Providing the `this` pointer enables the lambda function to access the `semaphore` object's scope.
    - `mutex` vs. `semaphore(1)`: `mutex`es are safer in the sense that only the thread which locks it , could unlock it. Implementation of semaphore uses a `mutex` and a `condition_variable_any`, so it does the same work as a `mutex`es, but it is an overkill when a simpler thing could solve that issue.

### 3.6.1 Reimeplement `eat()` in 3.5.3
- The original version of `eat()` in 3.5.3:
    ```C++
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
  ```
- Reimplement `eat()`: no longer need separate `waitForPermission()` and `grantPermission()` function implementations.
    ```C++
    static mutex forks[kNumForks];
    
    /* Strip out exposed size_t, mutex, and condition_variable_any,
       and replace the trio with a single semaphore object. */
    static semaphore numAllowed(kNumForks - 1);

    static void eat(unsigned int id) {
      unsigned int left = id;
      unsigned int right = (id + 1) % kNumForks;

      /* atomic --, it blocks if it attempts to decrement 0 */
      numAllowed.wait();

      forks[left].lock();
      forks[right].lock();
      cout << oslock << id << " starts eating." << endl << osunlock;
      sleep_for(getEatTime());
      cout << oslock << id << " all done eating." << endl << osunlock;

      /* atomic ++, no block, possibly unblocks other waiting threads */
      numAllowed.signal();

      forks[left].unlock();
      forks[right].unlock();
    }
    ```
> For exams (KOB):<br>(1) State the many pros on this approach over the busy waiting approach we initially used to avert the threat of deadlock in 3.5.2.<br>(2) Can you think of any situations when busy-waiting might be the right approach?

## 3.7 Reader–writer problem (KOB)

> This problem describes a situation where threads are trying to access the same shared resource at one time. Also referred to as the *consumer-producer problem* in a more generic sense.<br>It is different from the dining philosophers problem, where threads are bidding resource instances in a directed-cyclic-graph pattern.

Say there is a webpage content generator - the writer, and a browser to parse and render webpage's content - the reader. There is a buffer, whose capacity is `8` characters, and there are `40` characters to transmit in total. Obviously, the content cannot be delivered in an one-time transmission.

- We want the writer and reader to work in synchronization, so that the reader is consuming characters from the buffer while the writer is populating it.
- We also want to make sure that **the writer is ahead of the reader** (so the reader fail at getting a character from the buffer), **but by not too much** (so the buffer won't overflow).
- Obviously, an intuitive solution is having two child threads - one for the writer and the other for the reader. The buffer is the resource the two threads are sharing, and there are `8` character slots in that buffer.
- To make sure they work in synchronization (**no race condition nor deadlock**), we can use `semaphore` classes to enforce it.
    - Before the writer can write a character to the buffer, it has to lock the intended character slot. If no slot is available (i.e. not been read yet), it has to wait until one becomes so.
    - Before the reader can read a character from the buffer, it has to lock the intended character slot. If no slot is available (i.e. not been written yet), it has to wait until one becomes so.
    - Since **there are two conditions that will put a thread into wait**, we have to use two `semaphore` objects, as opposed to only one.
        - One `semaphore` object is `full`, and its initial `value` is `0`, denoting there is 0 slot that are full at the beginning.
        - The other `semaphore` object is `empty`, and its initial `value` is `8`, denoting there are 8 slots that are empty at the beginning.
        > It's like two dogs, whose sweater are marked "r" and "w" respectively, are running along a road, given that<br>(1) if "r" catches up with "w", "r" will stop and wait for "w" to run for a while, so "w" is ahead of "r";<br>(2) if "w" is too far ahead of "r", "w" will stop and wait for "r" to run for a while, so "r" is not lagging too behind.

The main function:
```C++
/* main.cc */
#include ...
using namespace std;

static void writer();
static void reader();

int main() {
    /* spawn off two child threads */
    thread w(writer);
    thread r(reader);
    /* join the child threads - don't forget */
    w.join();
    r.join();

    return 0;
}
```
The implementation of the thread routines.
```C++
/* routines.cc */
#include ...
using namespace std;

static const size_t kNumTotalChars = 40;
static const size_t kBufferSize = 8;
static char buffer[kBufferSize];

semaphore full(0);
semaphore empty(kBufferSize); /* or pass in a smaller positive number */

void writer() {
    for (size_t i = 0; i < kNumTotalChars; i++) {
        /* lock one empty slot. If none is empty, wait */
        empty.wait();

        buffer[i % kBufferSize] = generateOneChar();

        /* increment the number of full slots by 1 */
        full.signal();
    }
}

void reader() {
    for (size_t i = 0; i < kNumTotalChars; i++) {
        /* lock one full slot. If none is full, wait */
        full.wait();

        char ch = buffer[i % kBufferSize];
        parseOneChar(ch);
        /* increment the number of empty slots by 1 */
        empty.signal();
    }
}
```
- If the argument passed into the constructor of the `empty` object is also `0`, then the program will be entrenched in a deadlock, since each of the two `semaphore` objects are waiting one another's `value` to become positive so it can decrement its own `value`.
    > It's like the two aforementioned dogs are waiting on each other to run, so none of them is able to make any progress.

## 3.8 `myth-buster`: an example for load balancer
- Gist: connects to all 32 `myth` machines on Stanford campus and asks each for the total number of processes being run by CS110 students.
    > This is part of what a [load balancer](https://en.wikipedia.org/wiki/Load_balancing_(computing)) does - get the process counts for each physical machine.
### 3.8.1 Sequential version: slow (~1 min)
```C++
/* Sequential: slow */
#include ...
using namespace std;

static unsigned short kMinMythMachine = 1;
static unsigned short kMaxMythMachine = 32;

static void compileCS110ProcessCountMap(
                const unordered_set<string>& cs110studentIDs,
                map<unsigned short, size_t>& counts) {
  for (unsigned short num = kMinMythMachine;
       num <= kMaxMythMachine;
       num++) {
    int numProcesses = getNumProcesses(num, cs110studentIDs);
    if (numProcesses >= 0) { /* -1 expresses networking failure */
      counts[num] = numProcesses;
      cout << "myth" << num << " has this many CS110-student processes: " 
           << numProcesses << endl;
    }
  }
}
```
Nothing interesting here. The program just traverse all the machines. It is understandably slow (~1 min), since communication over the Internet is slow.
### 3.8.2 Parallel version: faster (~9 sec)
> Parallelism can be implemented with multiprocessing or multithreading. We go for the latter here.

> When you `ssh` to a `myth` machine, you certainly don't like waiting ~1 minute for the load balancer to assign you a machine.
```C++
/* Multithreading: faster */
#include ...
using namespace std;

/* globals (in heap, not in stack) to share across threads */
const unordered_set<string> cs110studentIDs;
map<unsigned short, size_t> counts;
static mutex processCountMapLock;


static void countCS110Processes(unsigned short num, semaphore& s) {
  int numProcesses = getNumProcesses(num, cs110studentIDs);
  if (numProcesses >= 0) {

    processCountMapLock.lock(); /* write to the shared: lock it first! */
    processCountMap[num] = numProcesses;
    processCountMapLock.unlock(); /* done writing: unlock */

    cout << oslock 
         << "myth" << num << " has this many CS110-student processes: "
         << numProcesses << endl
         << osunlock;
  }

  /* the thread completes and returns its permission slip back */
  s.signal(on_thread_exit);
  /* s.signal(on_thread_exit) signals the semaphore ON the thread's
   * exit; s.signal() does so BEFORE the thread's exit,
   * but this thread may not be truely done yet.
   */
}

static unsigned short kMinMythMachine = 1;
static unsigned short kMaxMythMachine = 32;
static int kMaxNumThrds = 8; /* max. number of child threads at a time */

static void compileCS110ProcessCountMap() {
  vector<thread> threads;

  /* used to limit number of threads */
  semaphore numAllowed(kMaxNumThrds);

  for (unsigned short num = kMinMythMachine;
       num <= kMaxMythMachine;
       num++) {
    /* wait for a permission to create a new child thread */
    numAllowed.wait();
    threads.push_back(thread(countCS110Processes, num, ref(numAllowed)));
  }

  for (thread& t: threads) t.join(); /* remember to reap the zombies */
}
```
Also, the lines below ...
```C++
processCountMapLock.lock(); /* write to the shared: lock it first! */
processCountMap[num] = numProcesses;
processCountMapLock.unlock(); /* done writing: unlock */
```
... could be replaced with
```C++
lock_guard<mutex> lg(processCountMapLock); /* the constructor locks it */
processCountMap[num] = numProcesses;
/* the lock is unlokced when the "lg" object is destroyed */
```
> A good practice: limit the maximum number of threads allowed at a time, to prevent stack-overflowing the process, overwhelming the kernel's thread manager, or overwhelming the web server your program is connecting to.<br>The OS may also impose this limit - when threads are too many, thread creating requests will be denied.

> SIDE NOTE:<br>In fact, however, for `myth` and big networks like Google, FaceBook, Microsoft, etc., users are still not satisfied with this performance - they don't want to wait for a second! So a common practice is having the load balancer run the program periodically and pre-compute the assignment.

> C++ grammar in object-oriented multithreading: pass in a thread routine: say, inside a class `A`'s method function, you want to install `A` class's another method `void foobar(int number, semaphore &s)` as the thread routine, the function pointer can be pushed through a thread constructor in one of the three ways listed below:
```C++
 thread t([this, n, &sem] { this->foobar(n, sem); }); /* 1st way */

 thread t([this](int number, semaphore &s) {
    this->foobar(number, s)
 }, n, ref(sem)); /* 2nd way */

 thread t(&A::foobar, this, n, ref(sem)); /* 3rd way */

 /* The template function ref() explicitly tells the compiler
  * "I'm passing by reference, not by value!", as the compiler
  * does not do type-matching for "thread" class constructor's
  * variadic argument list.
  */
 /* mutex and semaphore objects are not copiable. Their copy-constructors
  * are explicitly deleted in their class declarations */
```
###### EOF
