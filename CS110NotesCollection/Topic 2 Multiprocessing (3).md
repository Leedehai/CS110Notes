# Topic 2 Multiprocessing (3)
<div id="author-signature">Leedehai</div>
Friday, April 21, 2017

## 2.5 Inter-process communication I: pipe
### 2.5.1 Pipe and `pipe()`
- *Pipe* is a one-way condulet that carries *byte stream*, from a file to another. The two ends of the condulent, i.e. the sending file and the receiving file, are represented by their file descriptors.
``` 
 ____          out       p   i   p    e       in         ____
|    |╲        ─────────────────────────────────        |    |╲
| R  └─| file  ◀───    0 1 0 1 1 0 1 0 1    ◀───  file  | W  └─|
└──────┘(recv) ───────────────────────────────── (send) └──────┘
```
- In POSIX's C extension, `dprintf()` can be used to write formatted strings to a file designated by a file descriptor (its usage is like `fprintf()`). Unfortunately, the designers seemed to have forgotten to write "dscanf( )".
- There is also a system call named `pipe()`. It takes in a integer array of length two, and fills the array with two newly created file descriptors, **the second file being able to send byte stream to the first**. It returns `0` on seccess. On error, `-1` is returned, and `errno` is properly set.
```
                 ┌───────┬───────┐ pipe() ┌────────┬────────┐
    int fds[2]   │       │       │   ==>  │read(R) │write(W)│
                 └[0]────┴[1]────┘        └[0]─────┴[1]─────┘             
```
- Now that the system call `fork()` faithfully duplicate everything, including file descriptors, we can use `pipe()` to set up a communication channel between the parent and child process.
    > Corresponding file descriptors in the parent and the child process **point to the same file** via a shared entry in the system-wide file entry table. So if the parent or the child writes to a sending file, both the parent and child can receive the data from the receiving file. So by `fork()`, there are 4 channel established.<br> However, note that once data is read from an receiving end, that part of data is gone - it cannot be read again from another receiving end (hence the name *stream*). So effectively, the parent and child could **race for the data** if they are both reading - which is usually not a good thing.<br> Normally, if you want a bidirectional communication mechanism between the parent and child, you would need to set up two pipes - one for downlink, one for uplink.
```
    .PID 101.. ..   .PID 102.. ..       ┌───┬───┐   ┌───┬───┐ file
    :   ┌───┬───────────┐       :       │ R │ W │   │ R │ W │ desriptor
        │   │   :   :   │               └─┬─┴─┬─┘   └─┬─┴─┬─┘ tables
    : ┌─▼─┬─┴─┐       ┌─▼─┬───┐ :         │   └──┬────│───┘  
      │ R │ W │ :   : │ R │ W │           └─┬────│────┘      
    : └─▲─┴───┘       └─▲─┴─┬─┘ :           │    │                    
        │       :   :   │   │          ..─┬─▽──┬─▽──┬─.. file entry    
    :   └───────────────┴───┘   :         │ <=pipe= │    table   
    .. parent ...   ... child ...      ..─┴out─┴──in┴─.. (system-wide)  

      two separate FD tables      child shares file entries with parent
```

### 2.5.2 Redirect a file descriptor: `dup2()`
- System call `dup2()` takes two file descriptors as arguments. It **de-associate**s the second FD and its corresponding file, and **redirect** the second FD to the file pointed to by the first FD.
    > SIDE NOTE:<br>If the second file descriptor does not exist before, then `dup2()` will create it, and associate it to the file which the first file descriptor points to.<br> Related system calls: `dup()`(takes only one FD as argument, returns a new FD that points to the same file) and `dup3()`(similar to `dup2()` but takes an option flag as the 3rd argument).
- On success, these system calls return the second FD (which was passed in as the second argument).  On error, `-1` is returned, and `errno` is properly set.
- A `dup2()` can be used to redirect the standard input/output stream. For example, if we want a process to stop reading from the console stream, but from a given file stream, represented by a file descriptor `fdIn`, the code is this:
    ```C
    dup2(fdIn, STDIN_FILENO); /* STDIN_FILENO = the stdin stream FD */
    ```
    Thereafter, any read request against the file descriptor STDIN_FILENO (typically `0`) will still succeed, but by pulling data from the given file, no longer from the console input stream as it did before.
    ```
    ┌────────────┬────────────┬────────────┐
    │STDIN_FILENO│    ...     │   fdIn     │ file decriptor table
    └──────┬─┬───┴────────────┴─────┬──────┘  (for a process)
           │ └────────────────────┐ │ 
           |        △             │ │       
           X  ─ ─ ─ ┘redirect     │ │
           |                      │ │                
       ┌───▼─────┬────────────┬───▼─▼──┐     
       │ console │    ...     │  given │     file entry table
       │  input  │            │  file  │       (system wide)
       └───┬─────┴────────────┴────┬───┘ 
       ┌───▼─────┬────────────┬────▼───┐     
       │ console │    ...     │  given │     vnode table
       │  input  │            │  file  │      (system wide)
       └─────────┴────────────┴────────┘     
    ```
    Then, we can close the original file descriptor `fdIn` by using `close(fdIn)`, so the process can only access the given file through file descriptor `STDIN_FILENO`.
- Note that the each process's file descriptor table is maintained by the kernel and resides in the kernel space (not in the process's memory space), and identified by the corresponding process's PID. If this process is overwritten by calling `execvp()`, its file descriptor table remains intact, and the newly-initiated program, which continues to use the old PID, continues to use that table.
    > That means, if a process redirect its `stdin` stream (FD = `0`, which normally should read from console) to another file, and this process is transformed into another program's process by calling `execvp()`, then the new program's FD `0` continues to point to the this file, not to the console input as it normally should.

> Pause and think for a while: `fork()`, `waitpid()`, `execvp()`, `pipe()`, `dup2() ` - for each of them:<br>a) what does it do (in one sentence)?<br>b) does it affect the calling process's memory space? If so, how?<br>c) does it affect the calling process's FD table? If so, how?

> In shell, you can use `<` and `>` to redirect standard input/output of a program that is called from a cammand line. For example:<br> `$ /bin/sh < Command.txt`<br>`$ echo This repo is for CS110 > Readme.txt`

### 2.5.3 An example: implementing `subprocess()`
Suppose we have an executive named `sort`, which reads in character strings (separated by newlines and ends by Ctrl+D) from its `stdin` and sorts those strings in an alphabetic order, and prints them to its `stdout`.

Now, we want `sort` to read hard-coded strings from a program `driver`, instead of from the console, and outputs the result to the console.
> "subprocess" is just another term for "child process".

#### 2.5.3.1 Analysis
- The program `driver` has two functions: `main()` and `subprocess()`, implemented in two C files.

- We let the `driver` process fork itself, creating an identical child process, and establish a pipe between the parent and the child. The child redirects its `stdin` stream from the console to the pipe's reading end and uses `execvp()` to call our program `sort`.

- The parent writes data to the pipe and the child (now it is `sort`'s process) receives it from its `stdin` file stream, which is connected to the pipe, and outputs the result through `stdout` stream to the console.

- We need to *seal* unused ends of the pip by close the corresponding FD using `close()`. It prevents accidental reading/writing which may interrupt the byte stream. But if the pipe end is associated with multiple FDs, then the pipe end remains open until all FDs are closed. More importantly, closing a pipe's writing-to end **sends an EOF signal** to the receiver, otherwise the receiver would keep waiting for more input (in console input, EOF is sent by a Ctrl+D).

- A key observation is that, after `execvp()`, the process's PID remains the same and its **file descriptor table continues to be effective**, though this process is completely taken by another program.

- Another note is that, though the `main()` function calls the `subprocess()` function, the two functions are still in one process! Functions are just for human - after compilation and linking, they become fused in one routine of instructions and is loaded into the text segment of a process's memory space.

An illustration of the pipe - it has four ends after `fork()`!
```                         
 parent      fds[1] (W) ─────────────────────▶ fds[0] (R)                 
                        ──────────┐    ┌─────▶  <sealed>      
                                  │    │                               
                                  │    │                               
                             ┌────│────┘                                  
                             │    │          ┌─▷ STDIN_FILENO
                             │    │          │         
 child        fds[1] (W) ────┘    └──────────▶ fds[0] (R)    
              <sealed>   ────────────────────▶ <closed>
```
An illustration of the processes:
```
          execvp()   "/bin/sh -c /usr/bin/sort"           reaped
subprocess  ┌──●~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~●─ ─ ○ 
            │  /bin/sh △ △ △     △ STDIN_FILENO(stdin)  
            │          : : : via :   ||                 
driver      │          : : : pipe: fds[1]                 
────────────┴──────────●─●─●─...─●───────(w a i t)────────────▶
          fork()       words[i]  EOF(Ctrl+D)
```
***
The full picture in the shell.
```
Entry in file entry table: process (FD value), ...
console input:   shell(0), driver(0), 
console output:  shell(0), driver(1), /bin/sh(1), /usr/bin/sort(1)
pipe's in file:  driver(fds[1])
pipe's out file: /bin/sh(0), /usr/bin/sort(0)
```
```
┴ fork ● event ○ reap                             △ stdout (to console)
                                                  :
/usr/bin/sort process   ┌++●**********************●*●- ○
                        │   △ △ △     △ stdin        
                        │   : : :     :  ||          
/bin/sh process ┌~~~●+++┴+++:+:+:+++++:++||++(wait)+++++●─ ○
                │   /bin/sh : : :     : stdin   
                │           : : : via :  ||              
                │           : : : pipe: fds[1]           
driver   ┌──●~~~┴~~~~~~~~~~~●~●~●~...~●~~~~~~~(w a i t)~~~~~●─ ○ 
         │  driver          words[i]  EOF                    
         │                                
shell ───┴─────────────────────────(w a i t)────────────────────────▶
```
#### 2.5.3.2 The code

```C
/* main.c */
typedef struct {
    pid_t pid;    /* the subprocess's pid */
    int supplyfd; /* this FD is the writing-to end of the subprocess */
} subprocess_t;

subprocess_t subprocess(const char *command);

int main() {
    /* create a subprocess (i.e. child), 
     * get the subprocess's pid and writing-to end */
    subprocess_t sp = subprocess("/usr/bin/sort"); 

    const char *words[] = { /* hard-coded words */
      "felicity", "umbrage", "susurration", "halcyon",
      "pulchritude", "ablution", "somnolent", "indefatigable"
    };

    /* write the words to the subprocess through the writing-to end */
    for (int i = 0; i < sizeof(words); i++) {
        /* note that "%s" format of a char* is the content of the string,
         * not the value of the pointer itself.
         */
        dprintf(sp.supplypid, "%s\n", words[i]);
    }

    /* close the writing-to end, 
     * effectively sending a Ctrl+D (EOF) to the receiver */
    close(sp.supplypid);

    /* wait on the child and reap it - don't make the child an orphan! */
    int status;
    pid_t pid = waitpid(sp.pid, &status, 0);
    return pid == sp.pid && WIFEXITED(status) ? WEXITSTATUS(status) : -1;
}
```
Another function. Note that a new function does NOT correspond to a new process.
```C
/* subprocess.c */
typedef struct {
    pid_t pid;    /* the subprocess's pid */
    int supplyfd; /* this FD is the writing-to end of the subprocess */
} subprocess_t;

subprocess_t subprocess(const char *command) {
    int fds[2];
    pipe(fds); /* setting up a pipe */
     
    /* create a subprocess, i.e. child process,
     * and construct a subprocess_t instance to return */
    subprocess_t sp = {fork(); fds[1]}; 

    if (sp.pid == 0) { /* for the child process */
      /* prevent the child from writing to the pipe */
      close(fds[1]);

      /* redirect the child's stdin FD to the pipe's reading end,
         and de-associate fd[0] from the reading end */
      dup2(fds[0], STDIN_FILENO); 
      close(fds[0]);

      /* now, execute the intended program (/bin/sh) by execvp() */
      /* since the called program inherits the caller process's FD table,
       * its stdin stream is connect to the pipe's reading end */
      char *argv[] = {"/bin/sh", "-c", (char *) command, NULL};
      execvp(argv[0], argv);

      /* if execvp() succeeds, then /bin/sh reads from the pipe, 
       * getting the words published by the parent */
      /* if execvp() succeeds, the child process won't reach this line */
    }

    /* if execvp() succeeds, the child process won't reach this line */
    
    close(fds[0]); /* prevent the parent from reading from the pipe */
    return sp;
}
```
> Note that if `execvp()` succeeds, then the rest of the code is NOT executed, as the calling process is completely overwritten and rebooted by another program.

###### EOF
