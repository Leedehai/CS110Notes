# CS110Notes
CS110 Tutorial at Stanford

Hosted [here](http://web.stanford.edu/~hhli/CS110Notes/CS110NotesGateway.html), but it will be taken down when I graduate.

```                             
           _____    _____   __   __    ___      
          / ____|  / ____| /_ | /_ |  / _ \     
         | |      | (___    | |  | | | | | |    
         | |       \___ \   | |  | | | | | |    
         | |____   ____) |  | |  | | | |_| |    
          \_____| |_____/   |_|  |_|  \___/  
```

Course title: Principles of Computer Architecture

Spring 2017, offered by J. Cain

Author: L. Haihong (@Leedehai)

Caveat: no warranty, but feedback on factual errors are welcome!

## Index
Introduction<br /><a href="Introduction.html" target="_blank">[HTML]</a><a href="Introduction.md" target="_blank">[Markdown]</a><br /><br />
Topic 1 Filesystem (1): `cp`, "the three tables", `find`, intro to system calls<br /><a href="Topic 1 Filesystem (1).html" target="_blank">[HTML]</a><a href="Topic 1 Filesystem (1).md" target="_blank">[Markdown]</a><br /><br />
Topic 1 Filesystem (2): file systems, layering, more on system calls<br /><a href="Topic 1 Filesystem (2).html" target="_blank">[HTML]</a><a href="Topic 1 Filesystem (2).md" target="_blank">[Markdown]</a><br /><br />
Topic 2 Multiprocessing (1): `fork()`, `waitpid()`, reaping<br /><a href="Topic 2 Multiprocessing (1).html" target="_blank">[HTML]</a><a href="Topic 2 Multiprocessing (1).md" target="_blank">[Markdown]</a><br /><br />
Topic 2 Multiprocessing (2): `execvp()`, "the trio", shell<br /><a href="Topic 2 Multiprocessing (2).html" target="_blank">[HTML]</a><a href="Topic 2 Multiprocessing (2).md" target="_blank">[Markdown]</a><br /><br />
Topic 2 Multiprocessing (3): pipes, redirection<br /><a href="Topic 2 Multiprocessing (3).html" target="_blank">[HTML]</a><a href="Topic 2 Multiprocessing (3).md" target="_blank">[Markdown]</a><br /><br />
Topic 2 Multiprocessing (4): signals, signal handlers, signal masking<br /><a href="Topic 2 Multiprocessing (4).html" target="_blank">[HTML]</a><a href="Topic 2 Multiprocessing (4).md" target="_blank">[Markdown]</a><br /><br />
Topic 2 Multiprocessing (5): paging, virtual memory, scheduler, context switch<br /><a href="Topic 2 Multiprocessing (5).html" target="_blank">[HTML]</a><a href="Topic 2 Multiprocessing (5).md" target="_blank">[Markdown]</a><br /><br />
Topic 3 Multithreading (1): POSIX C `pthread`, race conditions<br /><a href="Topic 3 Multithreading (1).html" target="_blank">[HTML]</a><a href="Topic 3 Multithreading (1).md" target="_blank">[Markdown]</a><br /><br />
Topic 3 Multithreading (2): C++ `thread`, `mutex`<br /><a href="Topic 3 Multithreading (2).html" target="_blank">[HTML]</a><a href="Topic 3 Multithreading (2).md" target="_blank">[Markdown]</a><br /><br />
Topic 3 Multithreading (3): dining philosophers problem, deadlocks, permission slips, `condition_variable_any`, `lock_guard`<br /><a href="Topic 3 Multithreading (3).html" target="_blank">[HTML]</a><a href="Topic 3 Multithreading (3).md" target="_blank">[Markdown]</a><br /><br />
Topic 3 Multithreading (4): `semaphore`, reader-writer problem, `myth-buster`<br /><a href="Topic 3 Multithreading (4).html" target="_blank">[HTML]</a><a href="Topic 3 Multithreading (4).md" target="_blank">[Markdown]</a><br /><br />
Topic 3 Multithreading (5): concurrency patterns, `ice-scream-parlor`<br /><a href="Topic 3 Multithreading (5).html" target="_blank">[HTML]</a><a href="Topic 3 Multithreading (5).md" target="_blank">[Markdown]</a><br /><br />
Topic 4 Networking (1): sever, client, sockets, server-side multithreading<br /><a href="Topic 4 Networking (1).html" target="_blank">[HTML]</a><a href="Topic 4 Networking (1).md" target="_blank">[Markdown]</a><br /><br />
Topic 4 Networking (2): HTTP, networking system calls and data structures<br /><a href="Topic 4 Networking (2).html" target="_blank">[HTML]</a><a href="Topic 4 Networking (2).md" target="_blank">[Markdown]</a><br /><br />
Topic 4 Networking (3): networking API, MapReduce<br /><a href="Topic 4 Networking (3).html" target="_blank">[HTML]</a><a href="Topic 4 Networking (3).md" target="_blank">[Markdown]</a><br /><br />
Topic 4 Networking (4): non-blocking I/O<br /><a href="Topic 4 Networking (4).html" target="_blank">[HTML]</a><a href="Topic 4 Networking (4).md" target="_blank">[Markdown]</a><br /><br />
Topic 4 Networking (5): event-driven programming<br /><a href="Topic 4 Networking (5).html" target="_blank">[HTML]</a><a href="Topic 4 Networking (5).md" target="_blank">[Markdown]</a><br /><br />
Appendix A - Principles of Systems Design: abstraction, modularity and layering, naming and name resolution, caching, virtualization, concurrency, client-sever request-response<br /><a href="Appendix A - Principles of Systems Design.html" target="_blank">[HTML]</a><a href="Appendix A - Principles of Systems Design.md" target="_blank">[Markdown]</a><br /><br />

## Lab solutions (maintained by instructor)
<a href="https://quip.com/eAVXA4yhptT2" target="_blank">Lab 1: File Systems and System Calls</a><br />
<a href="https://quip.com/7OgvALBfs5aJ" target="_blank">Lab 2: Multiprocessing and Unix Tools</a><br />
<a href="https://quip.com/kcXwA57XWqsQ" target="_blank">Lab 3: Parallel Programming</a><br />
<a href="https://quip.com/lz5aA09sPPbn" target="_blank">Lab 4: assign3 Redux and Threads</a><br />
<a href="https://quip.com/2L3bApNzKaNe" target="_blank">Lab 5: Read-Write Locks, Event Barriers</a><br />
<a href="https://quip.com/BmCVAmSAjeAt" target="_blank">Lab 6: Threads vs Processes</a><br />
<a href="https://quip.com/PIz9A7Zi2onc" target="_blank">Lab 7: ThreadPools and Networking</a><br />

KOB = "knock on the blackboard" = "this is important" (for the purpose of course reviewing, not necessarily for learning)

LICENSE: [Creative Commons Attribution Share Alike 4.0 (CC-BY-SA-4.0)](https://creativecommons.org/licenses/by-sa/4.0/legalcode), not applicable to lab solutions.