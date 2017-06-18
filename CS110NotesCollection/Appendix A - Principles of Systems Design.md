# Appendix A - Principles of Systems Design
<div id="author-signature">Leedehai</div>
Friday, June 2, 2017

> Computer systems: filesystems, operating systems, compilation systems, database management systems, networks, and many large software.

> This course is implementation-driven, but we have to step back and look at the bigger picture.

## A.1 Abstraction
Separating behavior from implementation, defining clean interface to a component or subsystem that makes using and interacting with a system much, much easier.

- You know POSIX C's `pthread` and C++'s `thread`, but you do not know how they are implemented. In fact, threading is not natively supported by C or assembly code, so they are actually built upon abstraction.
- You know sockets, but the complicated world of networking layers are hidden behind the scene.
- You know SQL languages (from CS145), but the implementation of a relational database managing system is not shown to users.
- You know `g++` and `gdb`, but how are they implemented is not known to the clients.

## A.2 Modularity and layering
Subdivision of a larger system into a collection of smaller systems, which themselves can be further subdivided. Layering is a form of modularity, which is the organization of several modules that interact in some hierachical manner, where each layer typically only opens its interface to the module above and beneath it.

- Compilation system `gcc`: preprocessor, lexer, parser, semantic analyzer, code generator.
- Filesystems: sectors, blocks, inodes, file names, pathnames.
- Internet: link layer, internet layer, transport layer, application layer.
- GUI framework: MVC - model-view-control.

## A.3 Naming and name resolution
Names provide a way to refer to system resources, and name resolution is a means for converting between human-friendly names to machine-oriented ones.

- Filesystems: file names, inode numbers, block numbers.
- Internet: domain names, IP addresses.

## A.4 Caching
A cache is a component that stores data so that requests can be handled more quickly.

- Computer: L1, L2 cache
- Web servers: e.g. `memcache`, used in Facebook, etc. to handle large amount of request from users.
- Web proxy servers and DNS servers have cache.

## A.5 Virtualization:
Abstraction that make manu hardware resources look like one, or make one look like many.

- AFS (Andrew File System): make file systems on many machines look like one uniform machine. AFS grafts many independent, networked file systems into one rooted at `/afs`. You descend into various subdirectories of `/afs` with, in principle, zero knowledge of where in the world those directories are stored. Stanford uses OpenAFS for Stanford to log into AFS.
- Multithreading, multiprocessing - make one thread or process look like many.
- Vitual machines - make one physical machine look like many.
- Load balancer of web servers - make many servers look like one.

## A.6 Concurrency
Run in real or pseudo parallelism. Multiprocessing, multithreading, signal and intercept handlers are all examples of concurrency.

- Some programming languages—Erlang comes to mind— are so inherently concurrent that they adopt a programming model that makes race conditions virtually impossible. (Other languages—I'm thinking of pure JavaScript—adopt the view that concurrency, or at least threading, is too complicated and error-prone to support).

## A.7 Client-server request-and-response
Request/response is a way to organize functionality into modules that have a clear set of responsibilities. We've already had some experience with the request-and-response aspect of this.

- HTTP, IMAP, NFS, AFS, DNS

###### EOF