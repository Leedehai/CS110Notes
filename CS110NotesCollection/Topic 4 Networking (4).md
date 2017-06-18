# Topic 4 Networking (4)
<div id="author-signature">Leedehai</div>
Monday, June 5, 2017

> There is no class meeting on Monday, May 29. The class meeting on Friday, June 2 is about system design principles, a topic that is not part of networking. See note page *Appendix A - Principles of Systems Design*.

> This note page touches upon the technique of *non-blocking I/O*, and uses its application in networking as an example.

## 4.8 Non-blocking I/O
### 4.8.1 "Fast" and "slow" system calls
- Some system calls are thought to be **fast** not in that they return quickly, but in that there is no blocking time. That is, they are **active on the CPU throughout their lifetime**, from being initiated to being terminated.
    - Examples: `getpid()`, `execvp()`, and `fork()`. Note that the latter two system calls are generally time-consuming - rewriting or replicating a process's virtual memory space is not a cakewalk, and there are many optimizations to speed them up - they are still "fast" because they don't introduce blocking time.
- Some other system calls, however, have the potential to **block** the calling thread or calling process. Their blocking time is undetermined - maybe milliseconds, minutes, days, or even years. Hence they fall in the category of **slow** system calls.
    - Examples: `waitpid()` (waits for child process's state change), `accept()` (waits for connection request), `read()` (waits for bytes).

### 4.8.2 Non-blocking system calls
- "Fast" system calls are naturally non-blocking. But some "slow" system calls are capable of being non-blocking as well.
- `waipid()`: by adding the option of `WNOHANG` to the `waitpid()` call, we let `waitpid()` to be non-blocking if there is no state change in child processes, so that the calling process can move on to do other stuff at the time which would be otherwise spent on idly waiting on the child process.
- `read()`: by adding the option of `O_NONBLOCK` to `open()` (to create an FD), we can make `read()` calls to this file to be non-blocking. In this case, if there is no data immediately available for reading (no `EOF` either), `read()` will immediately return `-1` with `errno` being set to `EAGAIN`.
- `accept()`: When a listening socket is marked as non-blocking, if there is no pending connection request, an `accept()` call on this socket will immediately return with `-1` with `errno` being set to `EAGAIN` or `EWOULDBLOCK` (depending on the OS implementation).

#### 4.8.2.1 Case study: non-blocking input - `read()`
- This is a server that emits an English letter every 0.1 sec. This emulates the time a server might have to take to generate a response, e.g. fetching an image from Facebook's database.
```C++
#include ...
using namespace std;

static const string kAlphabet = "abcdefghijklmnopqrstuvwxyz";
static const useconds_t kDelay = 1e5; /* 1e5 us is 0.1 sec */
static void handleRequest(int connSock);

static const unsigned short kPort = 41411;
int main(int argc, char *argv[]) {
  /* create a listening socket */
  int server = createServerSocket(kPort);
  ThreadPool pool(128); /* we implemented ThreadPool in Assignment 6 */
  while (true) {
    /* wait and return a connected socket */
    int connSock = accept(server, NULL, NULL);
    pool.schedule([connSock]() { handleRequest(connSock); });
  }
  return 0;
}

void handleRequest(int connSock) {
  sockbuf sb(connSock);
  iosockstream ss(&sb);
  for (size_t i = 0; i < kAlphabet.size(); i++) {
    usleep(kDelay); /* wait for 0.1 sec */
    ss << kAlphabet[i] << flush;
  }
}
```
- This is a client that uses (traditional) blocking `read()`.
```C++
#include ...
using namespace std;

static const unsigned short kPort = 41411;

int main() {
  int clientSock = createClientSocket("localhost", kPort);
  size_t numSuccessfulReads = 0;
  size_t numBytes = 0;
  while (true) {
    char ch; /* a single-byte buffer */
    ssize_t count = read(clientSock, &ch, 1); /* potentially blocking */
    if (count == 0) break; /* break the loop on EOF */
    numSuccessfulReads++;
    numBytes += count;
    cout << ch << flush;
  }
  close(client);

  cout << endl;
  cout << "Alphabet Length: " << numBytes << " bytes." << endl;
  cout << "Num reads: " << numSuccessfulReads << endl;
  return 0;
}
```
Run the client - we discover that there are 26 `read()` calls:
```
$ ./blocking-alphabet-client
abcdefghijklmnopqrstuvwxyz
Alphabet Length: 26 bytes.
Num reads: 26
```
- This is anther client, which uses **non-blocking** `read()`.
```C++
#include ...
using namespace std;

static const unsigned short kPort = 41411;
int main() {
  /* send connection request to server */
  int clientSock = createClientSocket("localhost", kPort);
  setAsNonBlocking(clientSock); /* mark the SD as non-blocking */

  size_t numReads = 0;
  size_t numSuccessfulReads = 0;
  size_t numUnsuccessfulReads = 0;
  size_t numBytes = 0;
  while (true) {
    char ch;
    ssize_t count = read(client, &ch, 1);
    numReads++;
    if (count == 0) break; /* break the loop on EOF */
    if (count > 0) { /* read in something */
      numSuccessfulReads++;
      numBytes += count;
      cout << ch << flush;
    }
    else { /* didn't read in anything, return -1 as it's non-blocking */
      assert(errno == EAGAIN || errno == EWOULDBLOCK);
      numUnsuccessfulReads++;
    }
  }  
  close(client);

  cout << endl;
  cout << "Alphabet Length: " << numBytes << " bytes." << endl;
  cout << "Num reads: " << numReads << " (" << numSuccessfulReads << " successful, " << numUnsuccessfulReads << " unsuccessful)." << endl;
  return 0;
}
```
Run the client - we discover that there are 26 successful calls to `read()` and ~10<sup>7</sup> unsuccessful calls, because the server just emits one letter every 0.1 second:
```
$ time ./non-blocking-alphabet-client
abcdefghijklmnopqrstuvwxyz
Alphabet Length: 26 bytes.
Num reads: 11268991 (26 successful, 11268964 unsuccessful).
```
Here, `read()` is non-blocking, because the SD is marked as non-blocking.
> Data available? Expect `ch` to be updated and a return value of `1`.<br>No data available, ever (i.e. `EOF`)? Expect a return value of `0`.<br>No data available right now, but possibly in the future? Expect a return value of `-1` and `errno` to be set to `EAGAIN` or `EWOULDBLOCK`.
- For entirety, the implementation of `setAsNonBlocking()` is here - Linux gobbledygook.
```C++
#include ...
using namespace std;

/* mark the SD as non-blocking */
void setAsNonBlocking(int descriptor) {
  int flags = fcntl(descriptor, F_GETFL);
  if (flags == -1) flags = 0; /* if fcntl() fails, just go with 0 */
  fcntl(descriptor, F_SETFL, flags | O_NONBLOCK); /* retain other flags */
}
```
#### 4.8.2.2 Case study: non-blocking input/output
- The `OutboundFile` class is designed to read a local file and push its contents out over a supplied descriptor, and to do so without ever blocking - it does not block on reading from `source`, and it does not block on writing to `sink`.
```C++
class OutboundFile {
 public:
  OutboundFile();
  void initialize(const string &source, int sink);
  bool sendMoreData();

 private: /* unimportant for this lecture */
  int sink, source;
  static const size_t kBufferSize = 128;
  char buffer[kBufferSize];
  size_t numBytesAvailable, numBytesSent;
  bool isSending;

  bool dataReadyToBeSent() const;
  void readMoreData();
  void writeMoreData();
  bool allDataFlushed();
};
```
- The `initialize()` method identifies what local file should be used as a source of data and the descriptor (`sink`) into which that data should be written verbatim.
- The `sendMoreData()` method pushes as much data as possible to the supplied `sink`, without blocking. It returns `true` if it's at all possible there's more data to be sent, and `false` if all data has been fully pushed out.
- The unit test is this - it's a simple program prints the source code of itself to standard output:
```C++
/**
 * File: outbound-file-test.cc
 * ---------------------------
 * Demonstrates how one should use the OutboundFile class
 * and can be used to confirm that it works properly.
 */
#include "outbound-file.h"

int main() {
  OutboundFile obf;
  obf.initialize("outbound-file-test.cc", STDOUT_FILENO);
  while (obf.sendMoreData()) {;}
  return 0;
}
```
- A larger example of application: implementing a nonblocking server that happily serves up a copy of the server code itself to all clients. This server utilizes non-blocking I/O, instead of multithreading as in 4.3, to handle client connections.
    > Many company uses non-blocking I/O instead of multithreading because the latter is often error-prone - race conditions and deadlocks caused by carelessness in programming.
```C++
#include ...
using namespace std;

static const unsigned short kDefaultPort = 12345;
static const string kFileToServe("expensive-server.cc");
int main(int argc, char *argv[]) {
  int serverSocket = createServerSocket(kDefaultPort);
  if (serverSocket == kServerSocketFailure) {
    cerr << "Could not start server.  Port " << kDefaultPort << " is probably in use." << endl;
    return 0;
  }

  setAsNonBlocking(serverSocket);
  cout << "Static file server listening on port " << kDefaultPort << "." << endl;
  list<OutboundFile> outboundFiles;
  size_t numConnections = 0;
  size_t numActiveConnections = 0;

  while (true) {
    int clientSocket = accept(serverSocket, NULL, NULL);
    if (clientSocket == -1) {
      assert(errno == EAGAIN || errno == EWOULDBLOCK);
    }
    else { /* captured a connection request */
      OutboundFile obf;
      obf.initialize(kFileToServe, clientSocket);
      outboundFiles.push_back(obf);
      cout << "Connection #" << ++numConnections << endl;
      cout << "Queue size: " << ++numActiveConnections << endl;
    }

    /* manually multi-plexing: send data to clients, piece by piece */
    auto iter = outboundFiles.begin();
    while (iter != outboundFiles.end()) {
      if (iter->sendMoreData()) {
        ++iter;
      }
      else { /* no more data to send */
        iter = outboundFiles.erase(iter);
        cout << "Queue size: " << --numActiveConnections << endl;
      }
    }
  }
}
```
- As you see, if we use non-blocking I/O to handle multiple clients simultaneously, we have to write the multiplexing code ourselves - **a drawback of non-blocking I/O** compared to multithreading, where multiplexing is handled by the scheduler of OS.
- Also note that this server implementation contains **busy-waiting** and thus wastes CPU time. This will be addressed in 4.9 (event-driven programming).
- For your curiosity, the implementation of `OutboudFile` class is at the end of this note document.

## Appendix: implementation of `OutboudFile` class
The `OutboundFile` class is designed to read a local file and push its contents out over a supplied descriptor, and to do so without ever blocking.
```C++
/**
 * File: outbound-file.cc
 * ----------------------
 * Presents the implementation of the OutboundFile class.
 */

#include ...
using namespace std;

OutboundFile::OutboundFile() {
  isSending = false;
}

void OutboundFile::initialize(const string& source, int sink) {
  this->source = open(source.c_str(), O_RDONLY | O_NONBLOCK);
  this->sink = sink;
  setAsNonBlocking(this->sink);
  numBytesAvailable = numBytesSent = 0;
  isSending = true;
}

bool OutboundFile::sendMoreData() {
  if (!isSending) return !allDataFlushed();  
  if (!dataReadyToBeSent()) {
    readMoreData();
    if (!dataReadyToBeSent()) 
      return true;
  }  
  writeMoreData();
  return true;
}

bool OutboundFile::dataReadyToBeSent() const {
  return numBytesSent < numBytesAvailable;
}

void OutboundFile::readMoreData() {
  /* change all that by getting a chunk of readily available data
   * from (blocking) local file */
  ssize_t incomingCount = read(source, buffer, kBufferSize);
  if (incomingCount == -1) {
    assert(errno == EWOULDBLOCK);
    return;
  }

  numBytesAvailable = incomingCount;
  numBytesSent = 0;
  if (numBytesAvailable > 0) return;
  
  close(source);
  if (isSocketDescriptor(sink)) shutdown(sink, SHUT_WR);
  else setAsBlocking(sink);
  isSending = false;
}

void OutboundFile::writeMoreData() {
  auto old = signal(SIGPIPE, SIG_IGN);
  ssize_t outgoingCount = write(sink, buffer + numBytesSent, 
                                numBytesAvailable - numBytesSent);
  signal(SIGPIPE, old);
  if (outgoingCount == -1) {
    if (errno == EPIPE) {
      isSending = false;
    }
    else {
      assert(errno == EWOULDBLOCK);
    } 
  }
  else {
    numBytesSent += outgoingCount;
  }
}

bool OutboundFile::allDataFlushed() {
  bool allBytesFlushed;
  if (isSocketDescriptor(sink)) {
    assert(isNonBlocking(sink));
    ssize_t count = read(sink, buffer, sizeof(buffer));
    allBytesFlushed = count == 0;
  }
  else {
    assert(isBlocking(sink));
    int numOutstandingBytes = 0;
    ioctl(sink, SIOCOUTQ, &numOutstandingBytes);
    allBytesFlushed = numOutstandingBytes == 0;
  }
  
  if (allBytesFlushed) close(sink);
  return allBytesFlushed;
}
```

###### EOF