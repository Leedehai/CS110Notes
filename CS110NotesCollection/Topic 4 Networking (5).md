# Topic 4 Networking (5)
<div id="author-signature">Leedehai</div>
Wednesday, June 7, 2017

## 4.9 Event-driven programming
> *Event-driven programming* is a programming paradigm in which the flow of the program is determined by events such as user actions (e.g. mouse clicks, key presses), sensor outputs, or messages from other processes/threads, instead of by polling and busy-waiting.<br>In an event-driven application, there is generally a **loop that listens for events, and triggers a callback function when one of those events is detected**. Using hardware interrupts instead of a ceaselessly running loop also achieves this goal (recall your DSP course, huh?).<br>[Node.js](https://nodejs.org/en/about/) and most [GUI development tools](https://developer.apple.com/documentation/appkit) rely on event-driven programming.

> Event-driven programming is not just used in networking, but we will use networking as an example of its application scenario.

Discuss the epoll suite of functions: epoll_create, epoll_ctl, and epoll_wait.
Discuss the difference between edge-triggered and level-triggered events.
Implement an event-driven HTML server using nonblocking I/O in one process and one thread of execution that makes efficent use of the CPU.
### 4.9.1 Multiprocessing, multithreading v.s. non-blocking
-  Multiprocessing, multithreading
    - Pros: achieves parallelism; overcomes blocking system call's problem by allowing the CPU to do other things in the mean time.
    - Cons: concurrency issues - dealing with signal blocking/unblocking, avoiding race conditions and deadlocks.
- Non-blocking I/O:
    - Pros: manage multiplexing manually - agile; single process/thread, so no concurrency issues.
    - Cons: manage multiplexing manually - more work for programmer; busy-waiting, so CPU time is wasted.

### 4.9.2 Get rid of busy-waiting in non-blocking I/O
`epoll` is a library of Linux routines that help non-blocking servers yield the processor until it knows there's work to be done with one or more of the open client connections.
> `epoll` ("event polling") is available on Ubuntu, but may not be available on other Linux releases. `kqueue` is an alternative to `epoll`.
- `epoll_create()`, which creates something called a **watch set**, which itself is a set of file descriptors which we'd like to monitor. The return value is itself a file descriptor used to identify a watch set. Because it's a file descriptor, watch sets can contain other watch sets.
- `epoll_ctl()` is a control function that allows us to add descriptors to a watch set, remove descriptors from a watch set, or reconfigure file descriptors already in the watch set.
- `epoll_wait()` waits for I/O events on the watch set, blocking the calling thread until one or more events are detected.

This is the new server that utilizes non-blocking I/O to handle multiple clients but is without busy-waiting.
```C++
/**
 * File: efficient-server.cc
 * -------------------------
 * Simple application that relies on nonblocking
 * I/O and the suite of epoll functions to
 * implement an event-driven web-server.
 */

#include <iostream>
#include <map>
#include <string>
#include <unistd.h>                    /* for close() */
#include <sys/epoll.h>                 /* for epoll functions */
#include <sys/types.h>                 /* for accept() */
#include <sys/socket.h>                /* for accept() */
#include <string.h>                    /* for strerror */
#include "server-socket.h"             /* for createServerSocket() */
#include "non-blocking-utils.h"        /* for setAsNonBlocking() */
using namespace std;

/**
 * Function: buildEvent
 * --------------------
 * Populates the only two fields of an epoll_event struct that
 * matter to us, and returns a copy of it to the call site.
 * The events flag is really a set of flags stating what
 * event types and behaviors we're interested in for a file
 * descriptor, and fd is the file descriptor being registered.
 */
static struct epoll_event buildEvent(uint32_t events, int fd) {
  struct epoll_event event;
  event.events = events;
  event.data.fd = fd;
  return event;
}

/**
 * Function: acceptNewConnections
 * ------------------------------
 * Called when the kernel detects a read-oriented event
 * on the server socket (which is always in the watch set).
 *
 * In theory, many incoming requests (e.g. one or more) may have
 * accumulated in the time it took for the kernel to detect even the first one,
 * and because the server socket was registered for edge-triggered
 * event notification (e.g. via the EPOLLET bit being set), we are
 * required to accept each and every one of those incoming connections
 * before returning.  Each of those client connections needs to made
 * nonblocking and then added to the watch set (in edge-triggered mode,
 * initially for read events, since we need to ingest the request header
 * before responding).
 */
static void acceptNewConnections(int ws, int server) {
  while (true) {
    int clientSocket = accept4(server, NULL, NULL, SOCK_NONBLOCK);
    if (clientSocket == -1) return;
    struct epoll_event info = buildEvent(EPOLLIN | EPOLLET, clientSocket);
    epoll_ctl(ws, EPOLL_CTL_ADD, clientSocket, &info);
  }
}

/**
 * Function: consumeAvailableData
 * ------------------------------
 * Reads in as much available data from the supplied client socket
 * until it either would have blocked, or until we have enough of the
 * response header (e.g. we've read up through a "\r\n\r\n") to respond.
 * Because the client sockets were registered for edge-triggered read events,
 * we need to consume *all* available data before returning, else epoll_wait
 * will never surface this client socket again.
 */
static const size_t kBufferSize = 1;
static const string kRequestHeaderEnding("\r\n\r\n");
static void consumeAvailableData(int ws, int client) {
  static map<int, string> requests; // tracks what's been read in thus far over each client socket
  size_t pos = string::npos;
  while (pos == string::npos) {
    char buffer[kBufferSize];
    ssize_t count = read(client, buffer, kBufferSize);
    if (count == -1 && errno == EWOULDBLOCK) return; // not done reading everything yet, so return and expect to be called later
    if (count <= 0) { close(client); break; } // passes? then bail on connection, as it's borked
    requests[client] += string(buffer, buffer + count);
    pos = requests[client].find(kRequestHeaderEnding);
    if (pos == string::npos) continue;
    cout << "Num Active Connections: " << requests.size() << endl;
    cout << requests[client].substr(0, pos + kRequestHeaderEnding.size()) << flush;
    struct epoll_event info = buildEvent(EPOLLOUT | EPOLLET, client);
    epoll_ctl(ws, EPOLL_CTL_MOD, client, &info); // MOD == modify existing event
  }
  
  requests.erase(client);
}

/**
 * Function: publishResponse
 * -------------------------
 * Called on behalf of the specified client socket whenever the
 * kernel detects that we're able to write to it (and we're interested
 * in writing to it).
 *
 * This implementation should be more elaborate, but we can get away
 * with pretending the provided client socket is blocking instead of 
 * non-blocking because the string we write to it is so incredibly short.
 * A more robust implementation would check the return value to see how
 * much of the payload was actually accepted, keep calling write until
 * -1 was returned, etc.
 */
static const string kResponseString("HTTP/1.1 200 OK\r\n\r\n<b>Thank you for your request! We're working on it! No, really!</b><br/><br/><img src=\"http://vignette3.wikia.nocookie.net/p__/images/e/e0/Agnes_Unicorn.png/revision/latest?cb=20160221214120&path-prefix=protagonist\"/>");
static void publishResponse(int client) {
  write(client, kResponseString.c_str(), kResponseString.size());
  close(client);
}

/**
 * Function: buildInitialWatchSet
 * ------------------------------
 * Creates an epoll watch set around the supplied server socket.  We
 * register an interested in being notified when the server socket is
 * available for read (and accept) operations via EPOLLIN, and we also
 * note that the event notificiations should be edge triggered (EPOLLET)
 * which means that we'd only like to be notified that data is available
 * to be read when the kernel is certain there is data.
 */
static const int kMaxEvents = 64;
static int buildInitialWatchSet(int server) {
  int ws = epoll_create(/* ignored parameter = */ kMaxEvents); // value is ignored nowadays, but must be positive
  struct epoll_event info = buildEvent(/* events = */ EPOLLIN | EPOLLET, /* fd = */ server);
  epoll_ctl(ws, EPOLL_CTL_ADD, server, &info);
  return ws;
}

/**
 * Function: runServer
 * -------------------
 * Converts the supplied server socket to be nonblocking, constructs
 * the initial watch set around the server socket, and then enter the
 * wait/response loop, blocking with each iteration until the kernel
 * detects something interesting happened to one or more of the
 * descriptors residing within the watch set.  The call to epoll_wait
 * is the only blocking system call in the entire (single-thread-of
 * -execution) web server.
 */
static void runServer(int server) {
  setAsNonBlocking(server);
  int ws = buildInitialWatchSet(server);
  struct epoll_event events[kMaxEvents];
  while (true) {
    int numEvents = epoll_wait(ws, events, kMaxEvents, /* timeout = */ -1);
    for (int i = 0; i < numEvents; i++) {
      if (events[i].data.fd == server) {
        acceptNewConnections(ws, server);
      } else if (events[i].events & EPOLLIN) { // we're still reading the client's request
        consumeAvailableData(ws, events[i].data.fd);
      } else if (events[i].events & EPOLLOUT) { // we've read in enough of the client's request to respond
        publishResponse(events[i].data.fd);
      }
    }
  }
}

/**
 * Function: main
 * --------------
 * Provides the entry point for the entire server.  The implementation
 * of main just passes the buck to runServer.
 */
static const unsigned short kDefaultPort = 33333;
int main(int argc, char **argv) {
  int server = createServerSocket(kDefaultPort);
  if (server == kServerSocketFailure) {
    cerr << "Failed to start server.  Port " << kDefaultPort << " is probably already in use." << endl;
    return 1;
  }
  
  cout << "Server listening on port " << kDefaultPort << endl;
  runServer(server);
  return 0;
}
```
###### EOF