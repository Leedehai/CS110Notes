# Topic 2 Multiprocessing (2)
<div id="author-signature">Leedehai</div>
Monday, April 17, 2017

## 2.4 Execute a program: `execvp()`
- `execvp()` belongs to the `exec` system call family that run executable files.
- Prototype: `int execvp(const char *file, char *const argv[]);`
- If `execvp()` is called by a process, it will stop the caller process, the file given by `file` will be loaded into the caller's memory space and **overwrite** the program there, and the process is **rebooted**. Then, `argv` will be passed to program's `main()` function and execution starts.
    > As a result, once the specified program file starts its execution, the original program in the caller's memory space is gone and is replaced by the new process. This act is termed *overlay*. In short, it is "**overwrite and reboot**".
- Note that the `argv` is the same as the main function parameters discussed in 1.1. Its first element is a `char *` points to the executable's name string, and the last is a `NULL` pointer.
- Though the memory of the original process that invokes `execvp()` is overwritten, the process itself continue to exist, verified by the fact that the kernel-maintained **process ID remains unchanged**, and it is **still the child process of its parent**.
- `execvp()` returns `-1` on failure (e.g. cannot find file, access denied). It **won't return on success**, however, since the original process is already overwritten, including the instruction of calling `execvp()` itself.
  > The two bullet-points above are of great significance, as we will rely on it to write code.
> SIDE NOTE:<br>In `execvp()`, "v" means arguments are passed in as an array (vector) of pointers, and "p" means using the [environmental variable](https://en.wikipedia.org/wiki/Environment_variable) `PATH` to find the specified executable file if the string pointed by the first argument doesn't contain a slash `/`.

> SIDE NOTE:<br> With `execvp()`, you can deal with off-the-shelf programs that are provided as binary executables, instead of in the form of source code.

### 2.4.1 An familiar example: the shell command `find`
In a terminal, a typical call to `find` is like this:
```
$ find /usr/include -name stdio.h -print
```
Note that, the following arguments "`/usr/include`", "`-name`", "`stdio.h`", "`-print`" are **not** directly fed into the program `find`, but is fed into the shell program, together with the argument "`find`".

In the shell program, arguments are passed to a `execvp()` call, like this:
```C
/* arguments passed to the shell program */
char *argv[] = {"find", "/usr/include", "-name", 
                "stdio.h", "-print", NULL};
execvp(argv[0], argv);
```
An illustration of what happens in the caller process's memory space:
```
┌───────────────────────────┐         ┌───────────────────────────┐  
│                           │         │                           │
:           kernel          :         :           kernel          :   
│                           │         │                           │
╠═══════════════════════════╣         ╠═══════════════════════════╣ 
:                           :         :                           :     
├───────────────────────────┤- - - - -├───────────────────────────┤    
│                           │         │ "find",...,"-print",NULL, │ stack
│           stack           │         │ argv                      │   
│                           │         │─ ─ ─ ─ ─ ─ ─│─ ─ ─ ─ ─ ─ ─│<-%rsp
│─ ─ ─ ─ ─ ─ ─│─ ─ ─ ─ ─ ─ ─│<-%rsp   │             ▼             │    
│             ▼             │         │                           │
│             ▲             │         │                           │
│─ ─ ─ ─ ─ ─ ─│─ ─ ─ ─ ─ ─ ─│         │             ▲             │ 
│           heap            │         │─ ─ ─ - - - ─│─ - - - ─ ─ ─│  heap
│─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│         │─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│   
│        bss, data,..       │         │        bss, data,..       │  
│─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│         │─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│ 
│       program text        │<-%rip   │      NEW program text     │  
│   (binary instructions)   │         │   (binary instructions)   │
├───────────────────────────┤- - - - -├───────────────────────────┤<-%rip
:                           :         :                           :
└───────────────────────────┘ 0x0     └───────────────────────────┘ 0x0 
       Before execvp() -> overwrite & reboot -> After execvp()      
```
### 2.4.2 "The trio": `fork()`, `waitpid()`, and `execvp()`
Since we know,
- `fork()` replicates the calling process (parent) to a new process (child), and returns to child's PID to the parent;
 - `waitpid()` blocks the calling process (parent) and waits on the child specified by its PID;
- `execvp()` replace the entire memory space of the calling process with a new program's context, but does not change the process's PID and parent-child relationship.

We can emulate a shell's behavior! The shell process fork itself, and let the child call `execvp()` to transform the child to another program's instance. The parent just waits on the child (no longer a replica of the parent) to terminate, and reap the zombie.

A nice illustration:
```
            execvp()      (another routine)            reaped
            ┌──●~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~●─ ─ ─○ 
     child  │  child overwritten                            
            │                                                
parent      │                                        
────────────┴──────────(w a i t)───────────────────────────▶
          fork()
```
> SIDE NOTE:<br>In modern OSes, `fork()` might not completely replicate the calling process; it might replicate the calling process *lazily*, that is, it only replicates the portion that is necessary. One of the reasons is that if the newly replicated process is to be gutted by an `execvp()` a few instructions later, there is no point in copying the whole original process.

#### 2.4.2.1 Example of the trio working together: `system`
We have a program called `system`. It uses the shell the execute whatever command passed in as the argument.

For example, if a program calls `system(make)`, then this program will stall, and the executable `make` will be invoked to perform a "make" (compile, link, ...) in the current directory.

The code is
```C
#include <stdio.h>
#include <unistd.h>
#include <stdbool.h>
#include <sys/wait.h>
#include "string.h"
#include "exit-utils.h"

/* static global: visible only in this file */
static const int kExecFailed = 1;

int mysystem(const char *command) {
  pid_t pid = fork();
  if (pid == 0) { /* for the child process */
    char *args[] = {"/bin/sh", "-c", (char *) command, NULL};
    execvp(arguments[0], args); /* transform child's entire process */

    /* if the child process reaches here, the execvp() failed */
    fprintf(stderr, "execvp failed to invoke this: %s.\n", command);
    exit(kExecFailed);
  }
  
  /* the code below are only for the parent */
  int status;
  waitpid(pid, &status, 0);
  if (WIFEXITED(status)) /* if the child terminates normally */
    /* use the child's exit status as the mysystem()'s return value */
    return WEXITSTATUS(status);
  else
    /* use the child's exit status to indicate the error */
    return -WTERMSIG(status);
}
```
Here is a simple unit test:
```C
#include ...

int mysystem(const char *command);
static const size_t kMaxLen = 2048; /* static global */

int main(int argc, char *argv[]) {
  char buf[kMaxLen];
  while (true) {
    printf("$$ ");
    fgets(buf, kMaxLine, stdin); /* read string to char array */

    if (feof(stdin)) break; /* if reaches EOF of stdin (as file), break */

    /* don't forget to replace the trailing '\n' */
    buf[strlen(buf) - 1] = '\0';

    printf("retcode = %d\n", mysystem(buf));
  }
  
  printf("\n");
  return 0;
}
```
You can compile it and name the program as `myshell`. Now you have your own version of shell. In the real shell's terminal, you call the program `myshell`, and do works in that (pseudo) shell terminal implemented by yourself. You can even call `myshell` again in that (pseudo) shell terminal...
> Just like the movie [Inception (2010)](https://en.wikipedia.org/wiki/Inception): a dream inside a dream...
```
                "/bin/sh -c make"    "/bin/sh -c ls"
                ┌~~●***********●─ ○  ┌~~●***********●─ ○
                │ /bin/sh            │ /bin/sh       
                │                    │                       reaped
           ┌─●~~┴~~~~(wait)~~~~~~~~~~┴~~~~(wait)~~~~~~~~~~●─ ─ ○
real       │  myshell                                completes       
shell      │                                                
───────────┴───────────────────(w a i t)────────────────────────────▶
```
> As a more familiar example, you can rename the executable `myshell` to "python", and re-implement `mysystem()` properly, then you have your own python interpreter`:)`
```
leedehai@host:~$ python
Python 2.7.12 |Anaconda 4.1.1 (x86_64)| (default, Jul  2 2016, 17:43:17)
>>> print "hello"
hello
>>> 40 + 2
42
>>>
```
> SIDE NOTE:<br> In a shell, if a command line is trailing by `&`, as in `sleep 10 &`, the process(es) created by it will run in background, without suspending the shell process as it would have done without `&`. Such process is called the *background process*, opposed to the *foreground process*.

#### 2.4.2.2 `simplesh`: a miniature shell not unlike the normal shells
```C
#include <stdlib.h>     // exit
#include <stdio.h>      // for printf
#include <stdbool.h>    // for bool, true, false
#include <string.h>     // for strchr, strcmp
#include <unistd.h>     // for fork, execve
#include <sys/wait.h>   // for waitpid
#include "exit-utils.h" // 

static const char *const kCommandPrompt = "simplesh>";
static const unsigned short kMaxCommandLength = 2048; // in characters
static const unsigned short kMaxArgumentCount = 128;

static const int kForkFailed = 1;
static const int kWaitFailed = 2;

static void pullRest() {
  while (getchar() != '\n');
}

static void readCommand(char command[], size_t len) {
  char control[64] = {'\0'};
  command[0] = '\0';
  sprintf(control, "%%%zu[^\n]%%c", len);
  while (true) {
    printf("%s ", kCommandPrompt);
    char termch;
    if (scanf(control, command, &termch) < 2) { pullRest(); return; }
    if (termch == '\n') return;
    fprintf(stderr, "Command shouldn't exceed %hu characters.  Ignoring.\n", kMaxCommandLength);
    pullRest();
  }
}

static char *skipSpaces(const char *str) {
  while (*str == ' ') str++;
  return (char *) str;
}

static int parseCommandLine(char *command, char *arguments[], int len) {
  command = skipSpaces(command);
  int count = 0;
  while (count < len - 1 && *command != '\0') {
    arguments[count++] = command;
    char *found = strchr(command, ' ');
    if (found == NULL) break;
    *found = '\0';
    command = found + 1;
    command = skipSpaces(command);
  }
  arguments[count] = NULL;
  return count;
}

// left in for debugging purposes
/* static */ void printCommandLineArguments(char *arguments[]) {
  int count = 0;
  while (*arguments != NULL) {
    count++;
    printf("%2d: \"%s\"\n", count, *arguments);
    arguments++;
  }
}

static bool handleBuiltin(char *arguments[]) {
  if (strcasecmp(arguments[0], "quit") == 0) exit(0);
  return strcmp(arguments[0], "&") == 0;
}

static pid_t forkProcess() {
  pid_t pid = fork();
  exitIf(pid == -1, kForkFailed, stderr, "fork function failed.\n");
  return pid;
}

static void waitForChildProcess(pid_t pid) {
  exitUnless(waitpid(pid, NULL, 0) == pid, kWaitFailed,
         stderr, "Error waiting in foreground for process %d to exit", pid);
}

int main(int argc, char *argv[]) {
  while (true) {
    char command[kMaxCommandLength + 1];
    readCommand(command, sizeof(command) - 1);
    if (feof(stdin)) break;
    char *arguments[kMaxArgumentCount + 1];
    int count = parseCommandLine(command, arguments, sizeof(arguments)/sizeof(arguments[0]));
    if (count == 0) continue;
    bool builtin = handleBuiltin(arguments);
    if (builtin) continue; // it's been handled, and backgrounding a builtin isn't an option
    bool isBackgroundProcess = strcmp(arguments[count - 1], "&") == 0;
    if (isBackgroundProcess) arguments[--count] = NULL; // overwrite "&"
    pid_t pid = forkProcess();
    if (pid == 0) {
      if (execvp(arguments[0], arguments) < 0) {
    printf("%s: Command not found\n", arguments[0]);
    exit(0);
      }
    }
    
    if (!isBackgroundProcess) {
      waitForChildProcess(pid);
    } else {
      // don't wait for child, and let it roll in the background,
      // but recognize that we currently aren't reaping any background
      // processes when they terminate, and that's something that
      // we need to fix in future iterations of the shell
      printf("%d %s\n", pid, command);
    }
  }
  
  printf("\n");
  return 0;
}
```
The assignment will be a more sophisticated shell.
###### EOF