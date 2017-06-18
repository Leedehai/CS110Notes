# Introduction
Monday, April 03,  2017

## 0.1 Prerequisites
- Programming languages: C, C++
- Tools: `gcc`, `g++`, `gdb`, `valgrind` (memory-leak checker), `mercury` (version control)
- Default environment: Unix, running on a x86 machine.
> A good reference: [Allison's CS107 Survival Guide](http://web.stanford.edu/~adyuen/107), updated March 2015.

> Learn your `gdb`! It'll be your friends in later assignments. That being said, another powerful debugging tool is `printf()` and `cout`. When you are printing debugging info, you'd better add "[DEBUG]" at the beginning, so they stand out and you can remove them from your code more quickly after debugging is done.

## 0.2 Contents in this course (in a rough picture)
### 0.2.1 Filesystems
* Linux and C libraries for file manipulation: `stat`, `struct` `stat`, `open`, `close`, `read`, `write`, `readdir`, `struct dirent`, file descriptors, regular files, directories, soft and hard links, programmatic manipulation of them, implementation of `ls`, `cp`, `cat`, etc.
* naming, abstraction and layering concepts in systems as a means for managing complexity, blocks, `inode`s, `inode` pointer structure, `inode` as abstraction over blocks, direct blocks, indirect blocks, doubly indirect blocks, design and implementation of a file system.
* additional systems examples that rely on naming, abstraction, modularity, and layering, including DNS, TCP/IP, network packets, databases, HTTP, REST, descriptors and `pid`s.
* building modular systems with simultaneous goals of simplicity of implementation, fault tolerance, and flexibility of interactions.
* basic understanding to system calls.

### 0.2.2 Multiprocessing
* introduction to multiprocessing, `fork`, `waitpid`, `execvp`, process ids, inter-process communication, context switches, user versus supervisor mode.
* protected address spaces, virtual memory, main memory as cache, virtual to physical address mapping.
* concurrency versus parallelism, multiple cores versus multiple processors, concurrency issues with multiprocessing.
* interrupts, faults, systems calls, signals, design and implementation of a simple shell.
* virtualization as a general systems principle, with a discussion of processes, RAID, load balancers, AFS servers and clients.

### 0.2.3 Threading and Concurrency
* sequential programming, VLIW concept, desire to emulate the real world with parallel threads, free-of-charge exploitation of multiple cores (two per `myth` machine, eight per `corn` machine, 24 per `barley` machine), pros and cons of `thread`ing versus `fork`ing.
* C++ `thread`s, `thread` construction using function pointers, blocks, functors, `join`, `detach`, race conditions, `mutex`, IA32 implementation of `lock` and `unlock`, spin-lock, busy waiting, preemptive versus cooperative multithreading, `yield`, `sleep_for`.
* condition variables, rendezvous and thread communication, `unique_lock`, `wait`, `notify_one`, `notify_all`, deadlock.
* semaphore concept and `classs emaphore` implementation, generalized counter, pros and cons of `semaphore` versus exposed condition variables, thread pools, cost of threads versus processes.
* active threads, blocked threads, ready thread queue, high-level implementation details of the thread manager, `mutex`, and `condition_variable_any`.
* pure C alternatives via `pthread`s, pros of `pthread`s over C++11 thread package.

### 0.2.4 Networking and Distributed Computing
* client-server model, peer to peer model, protocol as contract and permitted conversation, request and response as a way to organize modules and their interactions to support a clear set of responsibilities.
* stateless versus keep-alive connections, latency and throughput issues, `gethostbyname`, `gethostbyaddr`, IPv4 versus IPv6, `struct` `sockaddr` hierarchy of `struct`s, network-byte order.
* ports, socket file descriptors, `socket`, `connect`, `bind`, `accept`, `read`, `write`, simple echo server, time server, concurrency issues, spawning threads to isolate and manage single conversations.
* C++ layer over raw I/O file descriptors, pros and cons, introduction to `sockbuf` and `sockstream` C++ classes.
* HTTP 1.0 and 1.1, header fields, `GET`, `HEAD`, `POST`, complete versus chunked payloads, response codes, web caching and consistency protocols.
* IMAP, custom protocols, Dropbox and iCloud reliance on variations of HTTP.
* Non-blocking I/O, where normally slow system calls like `open`, `accept`, `read`, and `write` return immediately instead of blocking, `select`, `epoll_*` set of functions, `libev` and `libuv` open source libraries.
* MapReduce programming model, implementation strategies using multiple threads and/or processes, comparison to previous systems that do the same thing, but not as well.

## 0.3 The readings
* *Computer Systems: A Programmer's Perspective* by Bryant and O'Hallaron, either the 2nd or 3rd edition.
* *Principles of Computer System Design: An Introduction* by Jerome H. Saltzer and M. Frans Kaashoek.

## 0.4 The assignments
* All programming assignments are due at the stroke of midnight, and you'll always be given at least 7 days to complete any one of them.
* If you need to submit an assignment after the deadline, you still can.  But doing so places a cap on the maximum number of points you can get for that assignment, depending on how late you submit.
* The first assignment must be turned in by the published deadline, without exception.
    1. Filesystem - Six degrees of Kevin Bacon
    2. Filesystem - Unix filesystems
    3. Multiprocessing - All things multiprocessing
    4. Multiprocessing - stsh - Stanford shell
    5. Multithreading - News aggregation
    6. Multithreading - News aggregation take II (thread pool)
    7. Networking - HTTP web proxy and cache
    8. Networking - MapReduce


## 0.5 The exams
### 0.5.1 The midterm:
- Time: 13:30 – 14:50 (lecture time), 80 min., on **Friday, May 12**, 2017
- Venue: CEMEX Auditorium
- Content: the first 9 lectures of the course (everything up to and including our discussion of virtual memory and the OS scheduler on April 28, but no threading or related concurrency issues).
- **Closed book, closed note, closed electronic device**, but one double-sided 8.5"-by-11" cheat sheet is allowed.

### 0.5.2 The final
- Time: 15:30 – 18:30, 180 min., on **Monday, June 12**, 2017
- Venue: NVIDIA Auditorium or Gates B03
- Content: all lectures.
- **Closed book, closed note, closed electronic device**, but two double-sided 8.5"-by-11" cheat sheets are allowed.

## 0.6 Some suggestions on convenient commands
We will rely on Stanford's `myth` cluster for labs and assignments (You can configure your "[OpenAFS for Stanford](https://uit.stanford.edu/service/openafs)" account for further convenience).
- In the `~/.bashrc` file (or `~/.zshrc`, depending on your shell) on your local machine, add this line:
    ```
    alias myth='ssh [YOUR_SUNetID]@myth.stanford.edu'
    ```
    And then use the command `source ~/.bashrc` in the local shell (or `source ~/.zshrc`, etc.) to make it effective.

- In the `~/.bashrc` file on your `myth` machine, add the following lines:
    ```
    alias ll='ls -l'
    alias sanitycheck='/usr/class/cs110/tools/sanitycheck'
    alias submit='hg commit -m "submit"; /usr/class/cs110/tools/submit'
    ```
    And then use the command `source ~/.bashrc` in the `myth`'s shell to make them effective.

These aliases will save you a lot of time, especially if your are scrambling to submit your assignment in the last minute!

###### EOF