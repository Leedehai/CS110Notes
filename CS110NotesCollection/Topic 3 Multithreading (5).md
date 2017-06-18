# Topic 3 Multithreading (5)
<div id="author-signature">Leedehai</div>
Friday, May 19, 2017

## 3.9 Concurrency patterns
> There’s value in learning programming patterns — patterns such as linear search, or divide-and-conquer—that can be generalized and used in other programs. Similarly, it might be helpful to describe the common concurrency patterns managed by the `thread`, the `mutex`, and our custom `semaphore` (together with `conditional_variable_any`).  Hopefully, these descriptions can help you identify some of the ways `mutexe`s and `semaphore`s are used, and give you some insight into what options you have for solving different concurrency and threading cases:<br>Binary lock, Generalized counter, Binary rendezvous, Generalized rendezvous, Layered construction.

### 3.9.1 Binary lock
**Mechanism**: The `mutex` is used as a single-owner lock. When constructed, it is in an unlocked state and threads bracket critical regions with matched `lock()` and `unlock()` calls on the `mutex` object.

**Application**: This sort of concurrency pattern is used to lock down sole access to some shared state or resource that only one thread can be manipulating at any one time.

### 3.9.2 Generalized counter
**Mechanism**: A **semaphore** is used to track some resource instances, be it a number of empty buffers, full buffers, available network connection, or what have you. The `semaphore` is essentially an integer count, capitalizing on its atomic increment, decrement, and the efficient blocking when a decrement is levied against a zero.

The semaphore is constructed to the initial count on the resource (sometimes `0`, sometimes a positive integer — it depends on the situation). As a thread requires an instance of resource, it calls `wait()` on the `semaphore` object responsible for that set of resource instances, so that it can transactionally consume a resource instance upon waking from that wait. Other threads (possibly, but not necessarily the same thread that consumes) `signal()` this `semaphore` object when a new resource instance becomes available.

**Application**: This sort of pattern is used to efficiently coordinate shared use of a limited resource that has a discrete quantity.  It can also be used to limit throughput (such as in the Dining Philosophers problem) where unmitigated contention might otherwise lead to deadlock.

### 3.9.3 Binary rendezvous
**Mechanism**: The `semaphore` can be used to coordinate cross-thread communication.  Suppose thread `A` needs to know when thread `B` has finished some task before it itself can progress any further. Rather than having `A` repeatedly loop (i.e. busy wait) and check some global state, a binary rendezvous can be used to foster communication between the two.  The rendezvous `semaphore` object is initialized to `0`.  When **thread** A gets to the point that it needs to know that another thread has made enough progress, it can `wait()` on the rendezvous `semaphore` object.  After completing the necessary task, `B` will `signal()` this object.

If `A` gets to the rendezvous point before `B` finishes the task, it will block until `B`'s `signal()`. If `B` finishes the task first, it calls **signal()** on the semaphore object, efficiently recording that the task is done, and when `A` gets to the `wait()`, it will be sail right through it. This binary rendezvous `semaphore` obejct records the status of one event and only ever takes on the value `0` (not-yet-completed or completed-and-checked) and `1` (completed-but-not-yet-checked).

**Application**: This concurrency pattern is sometimes used to wakeup another thread (such as disk reading thread that should spring into action when a request comes in), or to coordinate two dependent actions (a print job request that can't complete until the paper is refilled), and so forth.

If you need a bidirectional rendezvous where both threads need to wait for the other, you can add another `semaphore` object in the reverse direction (i.e. the `wait()` and `signal()` calls inverted). Be careful that both threads don’t try to wait for the other first and signal afterwards, else you can quickly arrive at deadlock!

### 3.9.4 Generalized rendezvous
**Mechanism**: Generalized rendezvous is a combination of binary rendezvous and generalized counter, where a single `semaphore` object is used to record how many times something has occurred. For example, if thread `A` spawned `N` thread `B`s and needs to wait for **all** of them make a certain amount of progress (not necessarily exiting) before `A` itself advancing, a generalized rendezvous might be used.

The `semaphore` object is initialized to `0`.  When `A` needs to sync up with the others, it will call `wait()` on the `semaphore` object in a loop, one iteration step for each thread it is syncing up with. `A` doesn't care which specific thread of the group has finished - it just cares about whether **all** of `B` threads have finished. If `A` gets to `wait()` before the `B` threads have finished, it will block, waking to "count" each child as the child calls `signal()`, and eventually move on when all dependent threads have checked back. If all the `B` threads finish before `A` arrives at `wait()`, `A` will quickly decrement the already-incremented-for-many-times `semaphore` object, once for each **thread**, and move on without blocking. The current value of the generalized rendezvous `semaphore` object gives you a count of the number of tasks that have completed that haven't yet been checked, and it will be somewhere between `0` and `N` at all times.

**Application**: The generalized rendezvous pattern is most often used to regroup after some divided task, such as waiting for several network requests to complete, or blocking until all pages in a print job have been printed.

> As with the generalized counter, it’s occasionally possible to use `thread::join()` instead of `semaphore::wait()`, but that requires the child threads to fully exit to "notify" the parent thread, and that’s not always what you want (though if it is, then `thread::join()` is just fine).

### 3.9.5 Layered construction

Once you have the basic patterns down, you can start to think about how `mutex`es and `semaphore`s can be layered and grouped into more complex constructions. 

Consider, for example, the constrained dining philosopher solution in which a generalized counter is used to limit throughput, and `mutex`es are used for each of the forks. Another layered construction might be a global integer counter with a `mutex` lock and a binary rendezvous `semaphore` object that can do something similar to that of a generalized rendezvous `semaphore` object. As tasks complete, they can each lock and decrement the global counter, and when the counter gets to `0`, a single `signal()` can be sent by the last thread to finish.

The combination of `mutex` and binary rendezvous `semaphore` could be used to set up a "race": thread `C` `wait()`s for the first of threads `A` and `B` to `signal()`. threads `A` and `B` each compete to be the one who calls `signal()` on the rendezvous `semaphore` object. Thread `C` only expects exactly one `signal()`, so the `mutex` lock is used to provide critical-region access so that only the first thread calls `signal()`, but not the second.

## 3.10 Example: ice cream parlor
> A example in which multiple thread classes are involved - a possible situation in real-world programming!

> This example originates from CS107 in 1990s :)

> A nice-to-know: in C++, there is a template class named `atomic<>`, which ensures some operations on the instantiated object is transactional, i.e. this template class carries a `mutex` within it so you don't need to manually manage the lock. Click [here](http://www.cplusplus.com/reference/atomic/atomic/) for details.

This program simulates the daily activity in an ice cream store. The simulation’s actors are:

- the customers who buy ice cream cones,
- the clerks who make ice cream cones,
- the single manager who supervises,
- the single cashier who accepts payment from customers.

A different thread is launched for each of these actors.

### 3.10.1 Scenario description

**Customer**s: each customer orders a number of ice cream cones, waits for them to be made, gets in line to pay, and then leaves (terminates). Customers are in a big hurry and don’t want to wait for one clerk to make several cones, so each customer dispatches one clerk thread for each ice cream cone he/she orders. Once the customer has all ordered ice cream cones, he/she gets in line at the cashier and waits to pay, in FIFO order.  After paying, each customer leaves (terminates).

**Clerk**s: each clerk thread delivers exactly one ice cream cone. The clerk scoops up a cone and then has the manager take a look at it to make sure it is absolutely perfect. If the cone doesn't pass the manager's test, it is thrown away and the clerk makes another one. Once an ice cream cone is approved by the manager, the clerk hands that ice cream cone to the customer, and the clerk leaves (teminates).

**Manager**: the single manager sits idly until a clerk needs his or her freshly scooped ice cream cone inspected.  When the manager hears of a request for an inspection, the manager determines if it the cone is deliverable to the customer. The manager leaves done when all orders from all customers have been delivered.

**Cashier**: the cashier idles while there are no customers in line. When a customer is ready to pay, the cashier handles the bill. Once the bill is paid, that customer leaves. The cashier should handle the customers in a FIFO manner. Once all customers have paid, the cashier leaves (terminates).

### 3.10.2 Analysis

### 3.10.3 The code

There is a handout for it: Multithreading and Synchronization Redux.pdf. Maybe I will write notes on it if I have time..