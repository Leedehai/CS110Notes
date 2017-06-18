# Topic 4 Networking (1)
<div id="author-signature">Leedehai</div>
Friday, May 19, 2017<br>Monday, May 22, 2017

> *Networking*: enlisting the services of multiple machines.<br>Conceptually, multiple machines form a *local area network* (LAN) via *hub*s, and multiple LANs form a larger LAN via *bridge*s. Implementations of LAN include *Ethernet*, *Wi-Fi*, and *ARCNET*. <br>Multiple incompatible LANs form a larger interconnected network, called *internet*, via *router*s. The most widespread implementation of internet is the *global IP internet*, or referred to as the *Internet*.<br>*World Wide Web*, or simply the *Web*, is the content served up through the Internet.

## 4.1 Client-server, request-response

### 4.1.1 "Function calls"
In CS110 we have four types of "function calls", all of which can be framed in a client-server relationship: The *client* makes a *request*, and then the *server* makes a *response*.
- Traditional function call. For instance, `main()` calls `foo()`, then `main()` is the client requesting a service from the server `foo()` - `foo()` does something on `main()`'s behalf.
- System calls. The user function is the client, requesting a certain service, e.g. reading a file, from the OS, which acts as the server.
- Inter-process calls through pipes. For instance, in our `farm` assignment, the master process wants to factorize some integer numbers, so it requests the service from multiple subprocesses (`worker`s), each of which factorizes an integer on the master's behalf.
- Client-server in web communications. For instance, you type a search key in [bing.com](www.bing.com), and your web browser process, acting as the client, sends your search key via [HTTP](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol) protocol to a Microsoft's machine near Seattle, and a process on that machine, identified by your browser with a "virtual process ID" (204.79.197.200:80) and acting as the server, responds via HTTP with a HTML file which contains search results, together with CSS files, JavaScript files, images, etc.
    > This inter-process communication relationship is like a pipe, but on inter-machine level instead of on intra-machine level. The term for this special "pipe" is *socket*.

We can see that networking is essentially no different from what we have seen in previous client-server relationships.
```
        ┌──────────┐                 ┌──────────┐            
        │          │────request─────▶│          │            
        │  Client  │                 │  Server  │            
        │          │◀───response─────│          │            
        └──────────┘                 └──────────┘ 
        main()                       foo()                 ..function call
        user process                 operating system      ..system call
        proc. 1 on machine A         proc. 2 on machine A  ..pipe (x2)
        proc. on machine A           proc. on machine B    ..socket
```
> Here, a *server* is a conceptual entity, which is not to be confused with a *sever machine*.

> In the case of networking, the machine on which a (or multiple) network clients or servers is running is referred to as a *host*.
### 4.1.2 Web browsing in terminal
An experiment in your terminal, to see how request and response are made via network.
```
$ telnet www.bing.com 80                     [type and return]
Trying 204.79.197.200...
Connected to a-0001.a-msedge.net.
Escape character is '^]'.
GET / HTTP/1.1                               [type and return] ┐ request  
Host: www.bing.com                           [type and return] │ in
                                             [return again]    ┘ HTTP/1.1
HTTP/1.1 200 OK                                                ┐
Date: Mon, 29 May 2017 22:02:07 GMT                            │
Cache-Control: private, max-age=0                              │
Content-Type: text/html; charset=ISO-8859-1                    │ response
Set-Cookie: ...blah blah blah                                  │ in
P3P: ...blah blah blah                                         │ HTTP/1.1
...                                                            │
<!DOCTYPE html...                                              │
...blah blah blah                                              │
...</html>                                                     ┘
Connection closed by foreign host.
$
```
In the terminal session above, you made a HTTP request to the website, and a process on that website's machine response with a long text string, which could be interpreted by your web browser and rendered in a pictorial form.

### 4.1.3 CS110 time server
Actually, the response does not have to conform to HTTP protocol.
Run an executable that fires up a server:
```
$ cd /usr/class/staff/examples/networking
$ ./cs110_time_server
Server listening on port 12345
                            
```
In another terminal, we access that server. This time, we make no request, and the server automatically makes a response.
```
$ telnet myth15.stanford.edu 12345
Trying 171.64.15.56...
Connected to myth15.stanford.edu
Escape character is '^]'.
Mon May 29 00:02:01 2017                        <-- the server's response
Connection closed by foreign host.
$
```

## 4.2 Implementing a server
> Linux's default network implementation complies with the Internet Protocol (IP).
### 4.2.1 Operative metaphor
- Colloquially, server-side applications wait by the phone (at a particular phone extension), praying that someone — no, anyone! — will call.
- Formally, server-side applications create a server socket that listens to a particular port.
- On the server side, a *socket* is represented by an integer identifier, called *socket descriptor* (SD), essentially like a file descriptor associated with an IP address (think phone number) and port (think phone extension). A socket is **bidirectional** - equivalent to a pair of pipes with opposite directions.
- You should also think of the port number as a virtual process ID that is associated with the actual process ID of the server program on the host.

### 4.2.2 Your first server: time server
Our time server accepts all incoming connection requests and quickly publishes the time, regardless of what the client's request is.
```C++
#include ...
#include "socket++.h"
using namespace std;

static const int kServerStartFailure = 2;

/* our cusrtom port number, should be larger than 1024 */
static const short kPortNumber = 12345;

static void publishTime(int connectedSocket);

int main() {
  /* create a listening socket, with a designated port number */
  int listenSocket = createServerSocket(kPortNumber);

  if (listenSocket == kServerSocketFailure) {
    cerr << "Error: Could not start server on this port." << endl;
    cerr << "Aborting... " << endl;
    return kServerStartFailure;
  }
  else {
    cout << "Server listening on port " << kPortNumber << "." << endl;
  }

  while (true) {
    /* wait until a connection request arrives, then open an
     * connected socket descriptor, which is readable & writable */
    int connectedSocket = accept(listenSocket, NULL, NULL);

    /* make a response to that connection */
    publishTime(connectedSocket);
  }

  return 0;
}

void publishTime(int connectedSocket) {
  /* The first five lines here produce the full time string that
   * should be published. 
   * These lines represent more generally the server-side computation
   * produce a response. It could have been a static HTML page, a
   * Bing search result, an RSS XML document, an image, or a video.
   */
  time_t rawtime;
  time(&rawtime);
  struct tm *ptm = gmtime(&rawtime); /* Greenwich Mean Time Zone */
  char timeString[128]; /* should be big enough */
  strftime(timeString, sizeof(timeString), "%c\n", ptm);

  /* now, we write to the connected socket descriptor, 
   * just like writing to a traditional FD */
  size_t nBytesWritten = 0, nBytesToWrite = strlen(timeString);
  while (nBytesWritten < nBytesToWrite) {
    nBytesWritten += write(clientSocket,
                           timeString + nBytesWritten,
                           nBytesToWrite - nBytesWritten);
  }
  close(connectedSocket); /* close the connected socket FD */

  /* alternatively, the writing part could be written in a more C++ way:
   *    sockbuf sb(connectedSocket);
   *    iosockstream ss(&sb);
   *    ss << timeString << flush;
   * connectedSocket is automatically closed when the sockbuf 
   * object is destroyed.
   */
}
```
### 4.2.3 Writing to SD in a more C++ way
Alternatively, the writing part below...
```C++
size_t nBytesWritten = 0, nBytesToWrite = strlen(timeString);
while (nBytesWritten < nBytesToWrite) {
  nBytesWritten += write(clientSocket,
                         timeString + nBytesWritten,
                         nBytesToWrite - nBytesWritten);
}
close(connectedSocket);
```
...could be written in a more C++ way:
```C++
sockbuf sb(connectedSocket);
iosockstream ss(&sb);
ss << timeString << flush;
```
Note that the SD `connectedSocket` is automatically closed when the `sockbuf` object is destroyed. You should **not** close the SD manually if a `sockbuf` object owns it.

Also note that a `iosockstream` object is both **readable and writable**, as long as the SD, which is owned by the underlying `sockbuf` object, is readable and writable.

Classes `sockbuf` and `iosockstream` are not provided by C++ STL, but rather by a third-party open source library named "[sock++](http://members.aon.at/hstraub/linux/socket++/docu/socket++.html)", whose development seems to be terminated.

### 4.2.4 Listening SD and connected SD
- A listening SD (`listenSocket` above) is
    - only created once, and exists for the entire life time of the server.
    - is **neither** readable **nor** writable, its sole purpose being listening to a port and waiting for connection requests.
- A connected SD (`connectedSocket` above), on the contrary,
    - only lasts as long as the communication is needed. It is opened when the server process accepts a connection request, and it **should be closed** when that communication ends.
    - is readable and writable via system calls `read()` and `write()`, like a file descriptor. And the corresponding socket behaves like a file. 

### 4.2.5 Block & wait for connection requests : system call `accept()`
System call `accept()` accepts a connection request on a listening socket. Its prototype is
```C++
int accept(int listenfd, struct sockaddr *addr, socklen_t *addrlen);
```
- Returns: non-negative connected SD on success, `−1` on error.
- The argument `listenfd` is a listening SD that has been created by our custom function `createServerSocket()`, which utilizes 3 system calls: partially create a SD with `socket()`, bound the SD to a local address with `bind()`, and convert the SD to a listening SD with `listen()`.
- The argument `addr` is a pointer to a `sockaddr` structure. This structure is filled in with the address of the peer socket, as known to the communications layer. The exact format is determined by the socket’s address family (see `socket()` and the respective protocol man pages).
- The `addrlen` argument is a value-result argument: it should initially contain the size of the structure pointed to by addr; on return it will contain the actual length (in bytes) of the address returned. When addr is `NULL` nothing is filled in.
- If no pending connections are present on the queue, and the socket is NOT marked as non-blocking, `accept()` blocks until a connection is present.

A nice illustration:
```
INSIDE THE SERVER MACHINE:
                ┌───────────────────────┐                     │
(no connection  []listening SD          │                     ▼ time
   reqeust)     │ (accept() blocks)     │
                │                       │ a server process
                │                       │
                │                       │                
                └───────────────────────┘
                ┌───────────────────────┐            
╸╸╸connection╸╸▶[]listening SD          │            
     request    │ (accept() returns     │
                │  a connected SD)      │ a server process
                │               │       │
                []connected SD ◀┘       │            
                └───────────────────────┘
                ┌───────────────────────┐            
                []listening SD          │            
                │                       │
                │                       │ a server process 
    requests ─▶ │ (readable & writable) │
◀════socket════▶[]connected SD ◀─▶ R/W  │            
  ◀─ responses  └───────────────────────┘
```

## 4.3 Server-side multithreading
Networking and multithreading are a natural match.
- The work a server needs to do in order to meet the client's request might be time consuming — so time consuming that a sequential implementation, like 4.2.2's while-true loop, might handicap the server's ability to accept future requests.
- One solution: as soon as `accept()` returns a connected SD, we spawn a child thread to get the time consuming computation off from the main thread. The child thread can make use of a second processor or a second core, and the main thread can quickly move on to its next call to `accept()`.

### 4.3.1 Re-implement the time server
```C++
#include ...
#include "socket++.h"
using namespace std;

static const int kServerStartFailure = 2;

/* our cusrtom port number, should be larger than 1024 */
static const short kPortNumber = 12345;

static void publishTime(int connectedSocket);

int main() {
  /* create a listening socket, with a designated port number */
  int listenSocket = createServerSocket(kPortNumber);

  if listenSocket == kServerSocketFailure) {
    cerr << "Error: Could not start server on this port." << endl;
    cerr << "Aborting... " << endl;
    return kServerStartFailure;
  }
  else {
    cout << "Server listening on port " << kPortNumber << "." << endl;
  }

  /* ThreadPool: already done in our assignment 6 */
  ThreadPool pool(4); /* spawn 4 child threads as workers */
  while (true) {
    /* blocks & wait */
    int connectedSocket = accept(listenSocket, NULL, NULL);
    /* schedule, let the Threadpool object handle multithreading */
    pool.schedule([connectedSocket] { publishTime(clientSocket); });
  }

  return 0;
}

void publishTime(int connectedSocket) {
  /* Similar to 4.2.2, but need to be thread-safe */
  time_t rawtime;
  time(&rawtime);
  struct tm tm;
  gmtime_r(&rawtime, &tm);
  char timeString[128];
  strftime(timeString, sizeof(timeString), "%c\n", ptm);

  sockbuf sb(connectedSocket);
  iosockstream ss(&sb);
  ss << timeString << endl << flush;
} /* sockbuf's destructor will close connectedSocket */
```
> Note that with multithreading, your code needs to be **thread-safe**, i.e. avoid race conditions and deadlocks. Reiterated, your code of reading and writing against a SD needs to be thread-safe, so it is not merely copy-and-paste the mono-thread version.

## 4.4 Implementing a client: time_client
The description:
- The client connects, i.e. "rings" the service's phone (IP) at a particular "extension" (port), and waits for the server to "pick up" (accept).
- The client says nothing.
- The server speaks by publishing the current time into its own end of the connection (socket) and then hangs up (close the server's connected SD).
- The client reads the published text, publishes it to the console, and then itself hangs up (close the client's SD).
### 4.4.1 The code
```C++
#include ...
using namespace std;

int main() {
  /* send the server a connection request, return an SD upon sucess */
  int clientSocket = createClientSocket("myth7.stanford.edu", 12345);
  if (clientSocket == kClientSocketError) {
    cerr << "Time server could not be reached" << endl;
    cerr << "Aborting" << endl;
    return 1;
  }

  /* read from the SD */
  sockbuf sb(clientSocket);
  iosockstream ss(&sb);
  string timeline;
  getline(ss, timeline);
  cout << timeline << endl;

  return 0;
} /* clientSocket will be closed by sockbuf's destructor */
```
###### EOF