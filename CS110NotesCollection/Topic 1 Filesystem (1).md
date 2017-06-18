# Topic 1 Filesystem (1)
<div id="author-signature">Leedehai</div>
Wednesday, April 05, 2017<br>
Friday, April 07, 2017

> A *filesystem* is used to **control how data is stored and retrieved**. Without a file system, information placed in a storage medium would be one large body of data with no way to tell where one piece of information stops and the next begins. By separating the data into pieces and giving each piece a name, the **data is segmented and identified**. Some file systems are used on **local storage devices**; others provide file access via a **network protocol**. Implementations of a filesystem vary, such as ext3, ext4, FAT, NTFS, HFS, APFS, AFS, NFS..

## 1.1 An appetizer: `cp`
Let us dive into the terminal land. As we all know, the command `cp` is a directive that replicates files. The resulting file is such a good replica that `diff` yields no difference between the resulting file and the original one.

```
cp makefile makefile.copy
```

Users tend to treat directives like `cp` as magic directives which are intrinsic to the terminal. What is little known to the the general users is that many of them are actually C/C++ programs, the only difference being the executables reside in somewhere else (on Unix, `cp`'s executable typically resides in `/bin`). It is the same case for directives like `ls`, `mv`, `find`, `clear`, even shell built-ins like `cd` and `which`, and many others that can sit at the `argv[0]` position of a command line.

> SIDE NOTE:<br>
> Command-line: suppose a directive needs `n` trailing input arguments:<br> `executable_name input_arg_1 ... input_arg_n`
> ```
> argc = n + 1 (it includes the directive's name!)
> argv:
>    ┌─────────┬─────────┬───────┬─────────┬────────┐
>    │ argv[0] │ argv[1] │  ...  │ argv[n] │  NULL  │
>    └─────────┴─────────┴───────┴─────────┴────────┘
>         │         │                 │ 
>         ▼         │                 │
> "executable_name" ▼                 ▼ 
>             "input_arg_1" ...  "input_arg_n"            
> ```
> `argv` array contains `n + 2` pointers of type `const char *` (including the pointer to the executable's name string and the trailing `NULL` pointer).

### 1.1.1 Implementing `copy.c`
`copy.c` is our reimplementation of `cp`, which
- serves as an example of implementing shell commands,
- introduces the notion of *file descriptor*s,
- utilizes low-level *system call*s `open`, `read`, `write`, and `close`. On [POSIX](https://en.wikipedia.org/wiki/POSIX) systems and in C programming language, they are mostly accessible through head file `unistd.h`. A system call is a functions that **requests a service from the kernel** of the operating system, and it is implemented by the operating system, instead of by the an application or a programming language library.
>  A good document that lists all system calls specified by POSIX: [POSIX system calls](http://www.tutorialspoint.com/unix_system_calls/index.htm).

We are not going to use `FILE *` (in C) or `ifstream`/`ofstream` (in C++)...

- `FILE *` and `ifstream`/`ofstream` are the standard file input/output interface in C and C++, respectively, but they are actually wrappers of low-level *system call*s. 
- we are going to use *file descriptor*s, as file identification in low-level system calls.
- > SIDE NOTE:<br> "openning" a file - <br>
C prototype: `FILE *fopen(const char *filename, const char *mode)` <br>
C++ prototype : `void ifstream::open(const char* filename,  ios_base::openmode mode = ios_base::in)`

### 1.1.2 Pseudocode (KOB):
```
PROCEDURE copy(source_file_name, destination_file_name):
    open the source file;
    create the destination file;
    create a data buffer of n bytes;
    LOOP:
        read at most n bytes from the source file to the buffer;
        IF n == 0:
            BREAK;
        M = 0;
        DO WHILE M < n:
            write m bytes from the buffer to the destinaton file;
            M += m;
        END
    END
    close the source file;
    close the destination file;
END PROCEDURE
```
### 1.1.3 Inplementation:
```C
/* copy.c */
#include <stdbool.h>
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h> // POSIX system calls
#include <errno.h>
#include <sys/stat.h>
#include <sys/stat.h>

static const int kWrongArgumentCount = 1;
static const int kSourceFileNonExistent = 2;
static const int kDestinationFileOpenFailure = 4;
static const int kReadFailure = 8;
static const int kWriteFailure = 16;
static const int kDefaultPermissions = 0644; // "rw-r--r--"

int main(int argc, char *argv[]) {
  if (argc != 3) {
    fprintf(stderr, "%s <source-file> <destination-file>.\n", argv[0]);
    return kWrongArgumentCount;
  }
  
  int fdin = open(argv[1], O_RDONLY);
  if (fdin == -1) {
    fprintf(stderr, "%s: source file could not be opened.\n", argv[1]);
    return kSourceFileNonExistent;
  }
  
  int fdout = open(argv[2], O_WRONLY | O_CREAT | O_EXCL, kDefaultPermissions);
  if (fdout == -1) {
    switch (errno) {
    case EEXIST:
      fprintf(stderr, "%s: destination file already exists.\n", argv[2]);
      break;
    default:
      fprintf(stderr, "%s: destination file could not be created.\n", argv[2]);
      break;
    }
    return kDestinationFileOpenFailure;
  }

  char buffer[1024];
  while (true) {
    ssize_t bytesRead = read(fdin, buffer, sizeof(buffer));
    if (bytesRead == 0) break;
    if (bytesRead == -1) {
      fprintf(stderr, "%s: lost access to file while reading.\n", argv[1]);
      return kReadFailure;
    }

    size_t bytesWritten = 0;
    while (bytesWritten < bytesRead) {
      ssize_t count = write(fdout, buffer + bytesWritten, bytesRead - bytesWritten);
      if (count == -1) {
	fprintf(stderr, "%s: lost access to file while writing.\n", argv[2]);
	return kWriteFailure;
      }
      bytesWritten += count;
    }
  }

  if (close(fdin) == -1) fprintf(stderr, "%s: had trouble closing file.\n", argv[1]);
  if (close(fdout) == -1) fprintf(stderr, "%s: had trouble closing file.\n", argv[2]);
  return 0;
}
```
### 1.1.4 The code explained
```C
int fdin = open(argv[1], O_RDONLY);
```
The system call `open()` takes as input the file name and the mode. The mode is an "oflag" (open flag) - here the oflag `O_RDONLY` means "open for reading only". This function does not return a pointer to the file, but a **non-negative integer** referred to as *file descriptor* (if you have played around with Matlab, you may have seen this). If the call fails, it returns `-1` in stead.

```C
int fdout = open(argv[2], O_WRONLY | O_CREAT | O_EXCL, kDefaultPermissions);
```
Similarly, here we bring in the destination file. The file is also represented by a file descriptor, which is a non-negative integer. If the call fails, it returns `-1`. Since the mode (oflag) constants are perfect power of 2, multiple mode constants can be organized by bitwise-OR operations to combine their effects. Here,
- `O_WRONLY` stands for "open for writing only", 
- `O_CREAT` stands for "if the file doesn't exist, create the file" (note the final "e" is missing from the word "create"),
- `O_EXCL` means "if the file arealy exist, open( ) will fail" ("exculsively").

Finally, the third argument is the file permission (an octal number `0644` here).

```C
ssize_t bytesRead = read(fdin, buffer, sizeof(buffer));
```
Here we read **at most** `sizeof(buffer)` bytes from the source file (denoted by the file descriptor `fdin`) to a buffer pointed to by the pointer `buffer`. The system call `read` returns the actual number of bytes it reads in. If it reaches the end of the current file block (not neccessarily the end of file, as the hard drive may have to spin to another segment), it will return a number smaller than the specified number of bytes, even `0`. If it fails, it returns `-1`.

Note that the return type is `ssize_t`, which stands for "signed size type", usually implemented as an `int` type.

```C
ssize_t count = write(fdout, buffer + bytesWritten, bytesRead - bytesWritten);
```
Here we write **at most** `bytesRead - bytesWritten` bytes, starting from the memory address `buffer + bytesWritten`, to the destination file (denoted by the file descriptor `fdout`). The call returns the actual number of bytes it writes to the file, which may be less than the specified number of bytes (e.g. the hard drive may have to spin), and returns `-1` if the writing operation fail.

```C
close(fdin);
close(fdout);
```
`close()` is also a system call. Once the sesison is completed, you need to close the files. In this process, resources associated with the file is de-allocated, as a part of the de-allocation process, the association between the file descriptor (an integer) and the file is broken up.

If closing the file succeeds, it returns `0`; otherwise it returns `-1` and a variable maintained by the system `errno` is properly set to denote the reason of failure.

### 1.1.5 A brief summary of the four system calls (KOB)
```C
/* open */
int open(const char *filename, int oflag, ...); // 
  /**
   * "..." stands for variadic function, since open() may take in 
   * more than two arguments (e.g. file access permission)
   * oflag: O_RDONLY, O_WRONLY, O_CREAT, O_EXCL, ...
   * oflags are perfect power of 2.
   */
/* read */
ssize_t read(int fd, void *start, size_t nbyte);
/* write */
ssize_t write(int fd, void *start, size_t nbyte);
/* close */
void close(int fd);
```

## 1.2 File descriptors and "the three tables" (KOB)
Let's look at what happens behind the scene when you acess a file.

- To "open" a file, i.e. grant your current user process the access to a file, you need to (directly or indirectly) call the system call `open()` and pass to it information like file name and access mode. The OS kernel will search the disk for the file. If successful, the kernel grant the user process the access to the file and returns a non-negative integer, namely a file descriptor, to the user process.
- To perform file I/O, your process passes the file descriptor (a non-negative integer) to the OS kernel through a system call, and the kernel will access the file on behalf of the process. The process does not have direct access to the file or inode tables.

What does "the kernel grant the user process the access" mean? Answer: it means that
- once the kernel finds the specified file's metadata (stored in a data structure called *inode*, to be decribed later) on the disk, it will cache the metadata to a table maintained by the kernel **system-wide**, called *vnode table*, in memory.
- then, an entry in another table called *file entry table*, also maintained by the kernel **system-wide** in memory, points to this *vnode* entry. This *file table* entry also documents the file's access permission, the cursor offset (i.e. the byte position where the next read/write operation starts, originally at 0), the reference count (i.e. how many file descriptors points to this file entry), etc.
- lastly, an entry in the *descriptor table*, maintained by the kernel **per-process** in memory, is allocated and points to the file entry in the file entry table.
- The file descriptor table's first three entries, denoted by file descriptor `0`, `1`, and `2`, are reserved for a process's `stdin`, `stdout`, and `stderr` streams, respectively. This abstraction makes standard I/O appears to be a file I/O to processes. `stdin`, `stdout`, and `stderr` normally point to the console, but can be redirected to other files.
- when a user process wants to access the file, the kernel will travel through the three tables (the process's descriptor table - the kernel's file entry table - the kernel's vnode table - the file's inode on disk - the disk), fetch the data, and return to the user process.
- the "v" in "vnode" stands for "virtual", as in "virtual file system".

```
           ┌──────────────┐┌──────────────┐┌──────────────┐         
           │ process 1201 ││ process 1705 ││ process 1707 │      
           ├──────────────┤├──────────────┤├──────────────┤       
           │        ┌───┐ ││              ││              │     
processes  │int fd1 │ 3 │ ││       ┌───┐  ││       ┌───┐  │                     
           │        ├───┤ ││int fd │ 3 │  ││int fd │ 3 │  │                       
           │int fd2 │ 4 │ ││       └───┘  ││       └───┘  │                         
           │        └───┘ ││              ││              │    
           └──────────────┘└──────────────┘└──────────────┘
                                               USER SPACE (in memory)
 ───────────────────────────────────────────────────────────────────────
file descriptor table(s)                       KERNEL SPACE (in memory)
   pid 1201                  pid 1705               pid 1707            
  ┌───┬───┬───┬───┬───┬───┐ ┌───┬───┬───┬───┬───┐  ┌───┬───┬───┬───┬───┐   
  │   │   │   │   │   │   │ │   │   │   │   │   │  │   │   │   │   │   │   
  └───┴───┴───┴───┴───┴───┘ └───┴───┴───┴───┴───┘  └───┴───┴───┴───┴───┘       
   0   1   2   3│  4│  ..    0   1   2   3│  ..     0   1   2   3│  ..     
                │   └──────────────────┐  │                      │           
file entry      │                      │  │                      │
  table         ▼                      ▼  ▼                      ▼         
 ┌───────────────────┬───┬───────────────────┬───┬───────────────────┐      
 │mode: r--r--r--    │   │mode: -w-------    │   │mode: rw-------    │    
 │cursor offset: 0   │   │cursor offset: 0   │   │cursor offset: 0   │    
 │reference count: 1 │...│reference count: 2 │...│reference count: 1 │    
 │          ┌──────┐ │   │          ┌──────┐ │   │          ┌──────┐ │    
 │vnode ptr │      │ │   │vnode ptr │      │ │   │vnode ptr │      │ │    
 │          └───┬──┘ │   │          └──┬───┘ │   │          └───┬──┘ │    
 └──────────────┼────┴───┴─────────────┼─────┴───┴──────────────┼────┘    
                │                      └────────────────┐       │             
vnode table     ▼                                       ▼       ▼             
  ┌────────────────────┬────────────────────────┬────────────────────┐       
  │type: direcotry file│                        │ type: regular file │       
  │ reference count: 1 │          ...           │ reference count: 2 │       
  │    inode: 2702     │                        │    inode: 3301     │       
  └────────────────────┴────────────────────────┴────────────────────┘
  ────────────────────────────────────────────────────────────────────
                                                                DISK
  ┌────┬─────────────┬─────────────────────────────────────────────┐
  │ .. │ inode table |             ...payload data...              │
  └────┴─────────────┴─────────────────────────────────────────────┘
```
## 1.3 Another example: `find`
The shell built-in `find` is a powerful tool to look for a file. A typical usage of if is
```
$ find /Library/C++_includes -name stdio.h -print
/Library/C++_includes/standard_include/c++/4.2.1/tr1/stdio.h
/Library/C++_includes/standard_include/stdio.h
/Library/C++_includes/standard_include/sys/stdio.h
```
Now we are going to implement a simpler version of this, called `search`. Its usage is
```
$ search /Library/C++_includes stdio.h
/Library/C++_includes/standard_include/c++/4.2.1/tr1/stdio.h
/Library/C++_includes/standard_include/stdio.h
/Library/C++_includes/standard_include/sys/stdio.h
```
### 1.3.1 Pseudocode
Gist of algorithm: recursively search each subdirectory inside the target directory. Depth-first search (DFS).
```
// recursion
SUBPROCEDURE listMatches(string path, int length, string pattern);
  path += "/"; // extend the string "path"
  length++;
  dir = open_directory(path);
  LOOP:
    entry = read_directory_entry(dir);
    IF entry.name == "." OR entry.name == "..":
      CONTINUE;
    END
    path += entry.name; // extend string "path" with this file name
    
    st = get_file_stat(path); // get the file's status info
    IF st IS directory file: // DFS
      listMatches(path, length + string_length(entry.name), pattern);
    ELSE IF st IS regular file:
      IF entry.name == pattern:
        PRINT(path) // print the whole path of this file
      END
    END 
  END
END SUBPROCEDURE

// main: starting point
PROCEDURE search(string target_directory, string pattern):
  st = get_file_stat(target_directory); // get the file's status info
  IF st is NOT a directory:
    RETURN;
  END

  path = target_directory;
  length = string_length(path);
  listMatches(path, length, pattern); // the recursion call
END PROCEDURE
```

### 1.3.2 Implementation
```C
#include <stdbool.h>   // bool
#include <stddef.h>    // size_t
#include <stdio.h>
#include <stdlib.h>
#include <stdarg.h>    // va_list, etc.
#include <sys/stat.h>  // stat
#include <string.h>    // strlen, strcpy, strcmp
#include <dirent.h>    // DIR, struct dirent

static void listMatches(char path[], uint length, const char *pattern) {
  /* extend the path: char *dest = strcpy(char *dest, const char *src) */
  strcpy(path + length, "/");
  length++;

  DIR *dir = diropen(path);
  while (true) {
    struct dirent *de = readdir(dir);
    if (de == NULL)
      break;
    if (strcmp(de->d_name, ".") == 0 || strcmp(de->d_name, "..") == 0)
      continue;

    strcpy(path + length, de->d_name); /* extend the path string */
    struct stat st;
    lstat(path, &st);

    if(S_ISDIR(st.st_mode)) {
      listMatches(path, length + strlen(de->d_name), pattern); /* DFS */
    }
    else if (S_ISREG(st.st_mode)) {
      if (strcmp(de->d_name, pattern) == 0) {
        printf("%s\n", path);
      }
    }
  }
  dirclose(dir);
}

int main(int argc, char *argv[]) {
  assert(argc == 3);

  const char *targetDirectory = argv[1];
  const char *pattern = argv[2];

  struct stat st;
  lstat(targetDirectory, &st);
  assert(S_ISDIR(st.st_mode));

  char path[kMaxPathLength];
  strcpy(path, targetDirectory);
  size_t length = strlen(path);

  listMatches(path, length, pattern);

  return 0;
}
```
### 1.3.3 The code explained
```C
#include <sys/stat.h>  // stat
```
The system call `lstat` is a C `struct` type that stores a file's status info, including the file's inode number, mode, owner ID, group ID, modification time, etc.

As a basic implementation, `int stat(const char *path, struct stat *buf);` stats the file pointed to by `path` and fills in the buffer pointed to by `buf`, and returns `0` on success, `-1` on failure. `lstat()` is identical to `stat()`, except that if `path` is a symbolic link, then the status info of the link itself is extracted, not the file the link refers to, hence the name "link-sensitive stat".

```C
DIR *dir = diropen(path);
```
`diropen` takes as input the path name (`char *`), and returns a pointer of type `DIR *`, which points to a structure representing a directory stream. The returned pointer serves as a handler for that directory, similar to `FILE *`.

```C
struct dirent *de = readdir(dir);
```
The function `struct dirent *readdir(DIR *dirp)` returns a pointer to a structure representing a directory entry at the current cursor position in the directory stream specified by `dir`, and position the stream at the next entry. It shall return a `NULL` pointer upon reaching the end of the directory stream. On failure, it returns a `NULL` pointer and properly sets `errno` to indicate the reason.

```C
assert(argc == 3);
```
If the expression in the parenthesis pair evaluates to `TRUE`, `assert()` does nothing. Otherwise, it displays an error message on `stderr` and aborts program. It is not part of POSIX, but a genuine C library function.

```C
S_ISDIR(st.st_mode);
S_ISREG(st.st_mode);
```
To check if the file, whose status info is stored in `st` of `stat` type, is a directory file or a regular file. They take in the `mode` member of `stat` structure and return a boolean. A similar POSIX macro is `S_ISLNK()` (to check if it is a symbolic link). The first "S" stands for "status".
> SIDE NOTE:<br>
> In the output of shell command `ls -l`, the leftmost column indicates a file's type and access permission, as in `drwxr-xr--`. If the file is a regular file, the first character is `-`; a directory file, `d`; a symbolic link file, `l`.

> SIDE NOTE:<br>
> A *symbolic link* (also *symlink* or *soft link*) is the nickname for a file. This symbolic link file contains a reference to another file or directory in the form of an absolute or relative path and that affects pathname resolution.<br> Symbolic links **operate transparently** for many operations: programs that read or write to files named by a symbolic link will behave as if operating directly on the target file. However, they have the side effect of changing an otherwise hierarchical filesystem from a tree into a **directed graph**, which can have consequences.<br> It is different from a [*hard link*](https://en.wikipedia.org/wiki/Hard_link).
###### EOF