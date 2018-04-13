# Topic 1 Filesystem (2)
<div id="author-signature">Leedehai</div>
Friday, April 07, 2017<br>
Monday, April 10, 2017

## 2.4 Filesystems
### 2.4.1 An overview
We assume the storage device is a local hard drive here, for simplicity.
- A hard drive, be it a magnetic tape, a HHD, or a SSD, is conceptually segmented in linear *sector*s, or *block*s, of a fixed size (e.g. 512 B), and those sectors are indexed: 0, 1, 2, ... sectors are accessed by their indecies.
- Sector is the smallest unit for read and write operations. It is a trade-off for space efficiency and I/O speed (typically measured in MB/s), and sequential I/O is much faster than random I/O for an HHD.
- A file might occupy one or more sectors. A sector is occupied by at most one file.
- If a file occupies multiple sectors, those sectors might not be consecutive or in order.

Inode.
- A file's metadata, including its owner, type, access permission, size, create time, the sector numbers of sectors in which the file's payload resides in, can be accessed by a data structure stored on disk, called inode (the "i" stands for "index"). Note that the file's own name is NOT stored in its inode, but in its directory file's payload.
- A file has one and only one inode.
- A file's "size" refers to the file's payload size, without its inode.
- Apart from *regular file*s, a *directory* is also a type of file and has its own inode.
- An inode has a fixed size, typically implemented as a `struct` type in C/C++.
- Inodes are stored in a dedicated part on the disk, referred to as the *inode table*. A sector can contain multiple inodes.
- An inode instance is stored on disk, while a vnode instance is stored in memory and maintained by the OS kernel.

We will use [Unix v6](https://en.wikipedia.org/wiki/Version_6_Unix) filesystem (1975) as an example to elaborate how filesystems work.

### 2.4.2 Layering for the Unix filesystem
- A program can create a file with a user-chosen name, read and write the file's content (*payload*), and set and get the file's metadata (owner, type, access permission, last modification time, ...). Files can be organized into directories with user-chosen names, forming a *naming network*. File hierarchies in different storage devices can be grafted together ("mount") to from a even larger hierarchy.
- To support these operations, Unix filesystem provides many APIs, including `open()`, `read()`, `write()`, `close()`, `mkdir()`, `mount()`, etc.
- To implement these APIs, the designer employed the strategy of layering. The filesystem makes use of several hidden layers which bridge machine-oriented names (i.e. addresses) to user-friendly names (i.e. character strings).
> *Layering*: As a form of modularity, layering is a basic idea to design large-scale systems. Apart from filesystems, the implementation of compilers uses the same strategy. That is, break down the entire task into many layers (or steps) and tackle them individually. The design of [internet layers](https://en.wikipedia.org/wiki/OSI_model) also uses the same idea. <br> Pictorially, though layers are stacked vertically and steps are aligned horizontally to form a pipeline, the ideas are essentially the same. 
```
                     USER
          ┌────────────────────────┐
        ▲ │  Symbolic Link Layer   │ pathname ▶ pathname
        │ ├────────────────────────┤
user    │ │Absolute Path Name Layer│ absolute pathname ▶ inode#
friendly│ ├────────────────────────┤
        ▼ │    Path Name Layer     │ pathname ▶ inode#
          ├────────────────────────┤
interface │    File Name Layer     │ file name ▶ inode#
          ├────────────────────────┤
        ▲ │   Inode Number Layer   │ inode# ▶ inode
machine │ ├────────────────────────┤
friendly│ │       File Layer       │ index# ▶ block# (blocks form a file)
        │ ├────────────────────────┤
        ▼ │      Block Layer       │ block# ▶ block data
          └────────────────────────┘
                    MACHINE
```

### 2.4.3 The layers explained
#### 2.4.3.1 The block layer
- Storage devices are conceptually segmented into *block*s, or *sector*s, of a fixed size (typically 512B). A block is the smallest unit to be operated by the filesystem.
    > If the sector size is too small, the overhead to keep track of free and allocated blocks increases; if too large, it may waste space.

    > Technically, a *block* is a software abstraction of a physical *sector* in the device. For us, they are the same.

- The name of a block is a number, which belongs to a compact set of integers, typically indicating the offset from the device's beginning.
- The **name-mapping algorithm** for a block is simple: it takes as input a block number, and returns the block content (each bit's value).
- The storage device istself may employ some block-managing mechanism under the hood, e.g. handling bad blocks.
- **Name discovery**: the filesystem starts with a *super block*, which has a well-known name (`1`). The super block serves as an anchor for the information that keeps track of which blocks are in use and which are available for assignment.
    > Different implementations of Unix may use different representations for the list of free blocks. A possible layout is like this:

    ```
     0       1       2                                          n-1
    ┌───────┬───────┬───────────┬─────────────┬────────────────────┐
    │ Boot  │ Super │ Bitmap of │    Inode    │     Payload        │
    │ block │ block │free blocks│    table    │  ... blocks ...    │
    └───────┴───────┴───────────┴─────────────┴────────────────────┘
    ```
    (Boot block: stores a program that starts the OS)

#### 2.4.3.2 The file layer (KOB)
- The user's stuff may exceed the size of block (e.g. 512 B) and may grow or shrink over time. To support this, a file layer is introduced. A file is essentially a linear array of bytes.
- The filesystem uses *inode* as container to record which blocks belong to each file. Together with some other metadata about a file, an `inode` can be implemented like this:
```C
struct inode { /* one instance for a file */
  int block_numbers[N]; /* N = 8 */
  int fileSize_in_bytes;
  int type; /* regular file, directory file, or symbolic link file */
  ... /* other metadata */
}
```
```                              
 ◀────────────── Inode table ─────────────────▶   
          ◀─ block ──▶                                            
┌─────────┬──┬──┬────┬───────────────────────┐ 
│         │  │██│    │                       │  
└─────────┴──┴──┴────┴───────────────────────┘                          
  ┌────────────┴──────────────────┐                     
  ◇                               ◇            
  ┌───────────────────────────────┐            
  │Inode                          │            
  │                               │            
  │ type: regular file            │    Note: an inode just occupies
  │ permission: rwxr-x--x         │    a sliver of a block.       
  │ owner ID: Leedehai            │            
  │ reference count: 1            │    (You can examine inode's        
  │ last modification time: ..    │     content using shell        
  │ ...                           │     command "stat")
  │         index=0 1 ...       7 │            
  │ block numbers┌─┬─┬─┬─┬─┬─┬─┬─┐│            
  │              └─┴─┴─┴─┴─┴─┴─┴─┘│            
  └───────────────────────────────┘            
```
- The blocks occupoed by a file may or may not be consecutive or in order. Picking which block to store data is the task of the storage device's software implementation.
- The **name-mapping algorithm (for small files)**: it takes as input an inode instance (typically a pointer to the inode instance) and an block index integer (`0`,...,`N-1`), and returns the block number.
```
                          ┌─┬─┬───┐                
                          │ │█│   │  Inode (inside an  
                          └─┴┬┴───┘    inode block)     
                    ┌────────┴┬───────────────┐        
                    ▼ 1       ▼ 2             ▼ 8       
                ┏━━━━━━━┓ ┏━━━━━━━┓       ┏━━━━━━━┓    
                ┃payload┃ ┃payload┃  ...  ┃payload┃    
                ┃ block ┃ ┃ block ┃       ┃ block ┃    
                ┗━━━━━━━┛ ┗━━━━━━━┛       ┗━━━━━━━┛
Note: an inode just occupies a sliver of a block.
```
- The **name-mapping algorithm (for large files)**: if a file spans across more than `N` (=8) blocks, the filesystem uses a more sophisticated mechanism to store the block numbers: the first `N-1`(=7) blocks are *indirect block*s. They do not contain payload data, but block numbers (if a block is 512B in size, a block number is 2B, then this block contains 512/2=256 block numbers). The last block, i.e. the `N-1 `th (=8th), is a *doubly indirect block*, namely, the block that contain block numbers of indirect blocks. With this design, a file's payload data can be of `(N-1)x(512/2)x512+1x(512/2)x(512/2)x512` bytes. When `N`=8, it is about 32MB.
    > \* "Indirect" blocks: blocks that reside in payload block area on the disk, but does not actually contain payload data. Instead, they contain block numbers.<br>
    * "Direct" blocks: blocks that actually contain payload data.<br>
    * For large files, their payload blocks' numbers are **not** all stored in the inode table area on the disk.
```
                   ┌─┬─┬───┐                             
                   │ │█│   │  Inode (inside an          
                   └─┴┬┴───┘    inode block)             
             ┌────────┴┬───────────────┬─────────────┐    
             ▼ 1       ▼ 2             ▼ 7           ▼ 8      
         ┌───────┐ ┌───────┐       ┌───────┐     ┌───────┐  
         │ indi. │ │ indi. │  ...  │ indi. │     │d.indi.│  
         │ block │ │ block │       │ block │     │ block │   
         └───┬───┘ └───────┘       └───┬───┘     └───┬───┘   
    ┌────────┴────┐              ... ──┴───┐         ├────────────┐ 
    ▼   1...256   ▼                        ▼ 256     ▼   1...256  ▼  
┏━━━━━━━┓     ┏━━━━━━━┓                ┏━━━━━━━┓ ┌───────┐    ┌───────┐
┃payload┃ ... ┃payload┃                ┃payload┃ │ indi. │    │ indi. │
┃ block ┃     ┃ block ┃ ...  ...  ...  ┃ block ┃ │ block │ .. │ block │
┗━━━━━━━┛     ┗━━━━━━━┛                ┗━━━━━━━┛ └───┬───┘    └───┬───┘
    1            256                   ┌─────────────┤       ... ─┴┐    
                                       ▼   1...256   ▼             ▼ 256
                                   ┏━━━━━━━┓     ┏━━━━━━━┓     ┏━━━━━━━┓
                                   ┃payload┃ ... ┃payload┃ ... ┃payload┃
                                   ┃ block ┃     ┃ block ┃     ┃ block ┃
                                   ┗━━━━━━━┛     ┗━━━━━━━┛     ┗━━━━━━━┛
Note: - an inode just occupies a sliver of a block.
      - suppose a (doubly) idirect block can store 256 block numbers.
```
#### 2.4.4.3 The inode number layer
- The filesystem names each inode instance by an inode number, which belongs to a compact set of integer.
- The **name-mapping algorithm** takes in an inode number and returns an inode instance from the inode table.
- **Name discovery**: this layer is responsible for keeping track of which inode numbers are in use and which are free.
    > Different implementations of Unix may use different representations for the list of free inode numbers.

#### 2.4.4.4 The file name layer (the machine-user interface of the filesystem)
- The inode number is the internal representation of a file, but is inconvenient for human beings.
- To create a file, the file system allocates an inode (inside a block in the inode table area on disk), initializes its metadata, and **binds the proposed file name to that inode in some directory file**.
    > You see, the file's name string is **not** in this file's inode or payload data, but **in the payload data of the directory** in which this file resides.
- By default, newly-created files are added to the working directory. The current working directory is indicated by by an inode number `cwd`, which is maintained by the kernel by the environment variable `PWD`. System call `chdir()` can allow a process to set this varible and change working directory.
    > *Environment variable*s (introduced by Unix version 7 in 1979) are a set of dynamic named values that can affect the way running processes will behave on a computer. They are part of the environment in which a process runs. <br> 
    For example, a running process can query the value of the `TEMP` environment variable to discover a suitable location to store temporary files, or the `HOME` or `USERPROFILE` variable to find the home or profile directory owned by the user running the process.
- In Unix, **a directory is also a file**, whose **payload data is a map** that maps file names to the corresponding inode numbers. System call `mkdir()` creates a zero-length directory file and sets the inode data of that file. When new files (be they regular files, directories, or else) are added to this directory, the (file name, inode number) pair is written into this directory file's payload data.
    > Reiterated, a file's name string is **not** in this file's inode or payload data, but **in the payload data of the directory** in which this file resides.<br> But to determine a file's type, you have to dive into that file's inode data.
```                                                 
    CS110 (directory)                                                         
    ├─ Syllabus.txt (regular file)                                                    
    └─ Homeworks (directory)                                                     
       ├─ HW1 (directory)
       └─ HowToSubmit.txt (regular file)
     inode 1001                                                     
     (CS110's inode)            block 9105                         
    ─┌───────────────┬ ─        ┌──────────────────────────────────┐
     │type: dir.     │          │                                  │
     │permission:..  │          │"Syllabus.txt": inode 2007        │─ ┐
     │ref cnt:..     │─────────▶│"Homeworks":  inode 2011          │   
     │block#: 9105   │          │                                  │  │
    ─└───────────────┴ ─        └──────────────────────────────────┘   
┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘
│                               block 9397                                    
     inode 2007                 ┌──────────────────────────────────┐
│    (Syllabus.txt's inode)     │                                  │
    ─┌───────────────┬ ─     ┌─▶│Content..                         │
│    │type: reg.     │       │  │                                  │
     │permission:..  │       │  └──────────────────────────────────┘
├ ─▷ │ref cnt:..     │───────┤  block 9395                                    
     │block#: 9397,  │       │  ┌──────────────────────────────────┐
│    │        9395   │       │  │                                  │
    ─└───────────────┴ ─     └─▶│Content..                         │
│                               │                                  │
      inode 2011                └──────────────────────────────────┘
│     (Homework's inode)        block 9301                         
    ─┌───────────────┬ ─        ┌──────────────────────────────────┐
│    │type: dir.     │          │                                  │
     │permission:..  │          │"HW1": inode 1015                 │
└ ─▷ │ref cnt:..     │─────────▶│"HowToSubmit.txt": inode 1070     │
     │block#: 9301   │          │                                  │
    ─└───────────────┴ ─        └──────────────────────────────────┘
```
#### 2.4.4.5 The path name layer & the absolute path name layer.
- A file's path are organized in the form of "a/b/c" string. Unix uses a recursive procedure to resolve the file's path.
- A absolute path starts from the root directory `/`, and has the form of "/a/b/c" (the leading "/" indicates the root directory). The root directory's inode number is `1` (the first inode; note that inode number starts from number 1, not 0).
> Similar heirarchical mechanism is utilized in internet's domain name resolution.

#### 2.4.4.6 The symbolic link layer
- Not covered in class, but you may refer to [this page](https://en.wikipedia.org/wiki/Symbolic_link) if interested.

## 2.5 More words on system calls
- System calls request services from the OS kernel on the user's behalf. They are actually "wrappers" of kernel's routines. For Unix, these wrappers are part of the C standard library, or `libc`.
    > The emergence of [C programming language](https://en.wikipedia.org/wiki/C_(programming_language)) is intertwined with Unix.
- System calls are different from traditional function calls, in that the routines called by the system calls reside in a portion of the memory that is inaccessible by the user. Typically that portion of virtual memory (*kernel address space*) is at the high-address part of the entire virtual memory.
    > SIDE NOTE:<br>The threshold of user address space and kernel address space, namely the virtual memory split, depends on the kernel implementation and machine architecture. Typically, on a 32-bit machine running Linux, the split is: user address space - `0x00000000` to `0xbffffffff` (3GB), kernel address space - `0xc0000000` to `0xffffffff` (1GB). On Linux, you can examine the threshold in `/boot/`.
    
    > SIDE NOTE:<br>On most mordern processors, software, will only use virtual addresses, during its normal operation. This holds true for kernel and user processes. Address translation hardware (usually inside the CPU), often referred to as a *memory management unit*, or MMU, automatically translates between the virtual and physical addresses, using a look-up table stored in the main memory, maintained by the OS. <br>One reason to introduce virtual addressing mechanism is using the mechanism of address translation to restrict user processes' access to the memory.

- When a user process makes a function call, it populates certain registers in the CPU. If the function call is a system call, it will write the opcode and arguments to CPU registers and trigger a **software interuption**, transferring the control to the interruption handler function. The handler, which resides in the kernel address space, reads the register values to the kernel space and performs the task requested by the system call, and then populate the return value to CPU register `RAX` (here the CPU is in *kernel mode*, an elevated previledge mode higher than the normal *user mode*). After that, the user process takes back the control and resumes execution. This black box mechanism guarantees that the user process has no access to the kernel address space's content.
    > **Q**: what if system calls are implemented like normal function calls? <br>**A**: In normal function call, when user function `A()` calls on function `B()`, a chunk of memory **in the user space** will be allocated for `B()` (i.e. its stack frame). When `B()` completes, that memory chunk is released. However, though `B()`'s stack frame is released, it might not be erased immediately. Thus, `A()` can still **read the "ghost data" via pointer arithmetics**, potentially compromising data security. Also, `B()` can read `A()`'s stack frame as well. <br> Therefore, if `B()` is a kernel function and hence its data needs to be protected, `B()` needs to reside in the place `A()` has no access to - so `B()` needs to be placed in the kernel space, and one way to trigger `B()` is software interruption.
```
        ┌────────────────────────────────────────┐ 0xffffffff            
        │                                        │ 
        :                 kernel                 :                  
        │                                        │ 0xc0000000 
  - - - ╠════════════════════════════════════════╣ - - -         
        │                                        │ 0xbfffffff           
        :                                        :                     
        │                                        │            
      ▲ ├────────────────────────────────────────┤  
      │ │ user        main()'s stack frame       │          
      │ │ stack       ─  ─  ─  ─  ─  ─  ─  ─  ─  │
      │ │             foo()'s stack frame        │       
   o  │ │─ ─ ─ ─ ─ ─ ─ ─ ─ ─│─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │<- %esp            
   n  │ │                   ▼                    │   (stack pointer
   e  │ │─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │    register)
      │ │          memory mapped region          │ 
   u  │ │          for shared libraries          │ 
   s  │ │─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │ 
   e  │ │                   ▲                    │      
   r  │ │─ ─ ─ ─ ─ ─ ─ ─ ─ ─│─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │<- brk   
      │ │              runtime heap              │   (variable 
   p  │ │─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │    maintained       
   r  │ │                   bss                  │    by kernel)       
   o  │ │        (unitialized static data)       │           
   c  │ │─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │           
   e  │ │         rodata (read-only data)        │ ┐          
   s  │ │─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │ │          
   s  │ │                  data                  │ ├ a.out           
      │ │─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │ │           
      │ │              program text              │ ┘
      │ │         (binary instructions)          │ <- %rip   
      ▼ ├────────────────────────────────────────┤    (instruction pointer
        │                                        │     register points to
        :                                        :     the next instruct.)      
        │                                        │ 
        └────────────────────────────────────────┘ 0x00000000             
                   Virtual Memory Space                              
```
###### EOF
