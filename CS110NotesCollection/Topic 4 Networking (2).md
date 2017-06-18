# Topic 4 Networking (2)
<div id="author-signature">Leedehai</div>
Wednesday, May 24, 2017

## 4.5 A taste of HTTP: emulating `wget`
> [HTTP](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol): Hypertext Transfer Protocol, an application layer protocol widely used in today's web.

> Ha, I just used this command tool, `wget`, a week ago.

### 4.5.1 Description

`wget` is a command line utility that, given a [Universal Resource Locator](https://en.wikipedia.org/wiki/URL) (URL), can downloads files (.html, .js, .jpg, .mov, .zip, or whatever). For example, we want to download a picture of a guru in computer science, Professor [Donald Knuth](https://en.wikipedia.org/wiki/Donald_Knuth) at Stanford.
```
$ wget https://en.wikipedia.org/wiki/File:KnuthAtOpenContentAlliance.jpg -O DKnuth.jpg
--2017-05-28 08:07:15--  https://en.wikipedia.org/wiki/File:KnuthAtOpenContentAlliance.jpg
Resolving en.wikipedia.org... 198.35.26.96, 2620:0:863:ed1a::1
Connecting to en.wikipedia.org|198.35.26.96|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: unspecified [text/html]
Saving to: ‘DKnuth.jpg’

DKnuth.jpg                    [ <=>                                ]  44.27K  --.-KB/s    in 0.01s

2017-05-28 08:07:16 (3.71 MB/s) - ‘DKnuth.jpg’ saved [45337]
$
```
Our implementation, `web-get`, simplifies `wget`'s functionality, but nonetheless serves a good example to illustrate most basic parts of the HTTP protocol, since this tool need to speak HTTP, specifically HTTP/1.0 dialect, to fulfill its mission.
```
$ web-get https://en.wikipedia.org/wiki/File:KnuthAtOpenContentAlliance.jpg
Total number of bytes fetched: 45337
$
```
> SIDE NOTE:<br>Review from EE284 or CS144: Internet protocol suite in TCP/IP 4-layer model<br>Application layer (with typical port number) - BGP(179), **DHCP**(67,68), **DNS**(53), **FTP**(21), **HTTP**(80), **HTTPS**(443), **IMAP**(143), NTP(123), **POP3**(110), **SMTP**(25), **SSH**(22), **Telnet**(23), ...<br>Transport layer - **TCP**, **UDP**, DCCP, SCTP, RSVP, ...<br>Internet layer - **IP** (IPv4, IPv6), ICMP, ECN, IGMP, OSPF, ...<br>Link layer - **ARP**, NDP, PPP, **MAC** (Ethernet, DSL, ISDN, FDDI), ...

### 4.5.2 The code (KOB)
First and foremost, we can see the high-level structure from the `main()` and the `pullContent()`.
```C++
#include ...
#include "socket++.h"
using namespace std;

int main(int argc, char *argv[]) {
  pullContent(parseURL(argv[1]));
  return 0;
}

/* helper: skip "http://" and split the rest into "host" and "path" */
pair<string, string> parseURL(string url) {
  if (startsWith(url, kProtocolPrefix))
    url = url.substr(kProtocolPrefix.size());
  size_t found = url.find('/');
  if (found == string::npos)
    return make_pair(url, kDefaultPath); /* defined in <utility> */
  string host = url.substr(0, found);
  string path = url.substr(found);
  return make_pair(host, path);
}

/* helper: get the filename, i.e. the last component of the path */
string getFileName(const string& path) {
  if (path.empty() || path[path.size() - 1] == '/') {
    return "index.html"; /* not always correct, but not the point */
  }
  size_t found = path.rfind('/');
  return path.substr(found + 1);
}

/* Do the real work */
void pullContent(const pair<string, string>& parsedURL) {
  const string &host = parsedURL.first; /* www.google.com */
  const string &path = parsedURL.second;/* images/branding/(...).png */
  
  /* The six steps of downloading */
  /* step 1: connect to the server (port 80 for HTTP), get an SD */
  int clientSocket = createClientSocket(host, 80);
  /* step 2: layer a buffer over the "bare" SD */ 
  sockbuf sb(clientSocket);
  /* step 3: create a stream class around the buffer */
  iosockstream ss(&sb);
  /* step 4: send request via the socket stream */
  issueRequest(ss, host, path);
  /* step 5: skip the header of the response (placed in the stream) */
  skipHeader(ss);
  /* step 6: save the payload (placed in the stream) */
  savePayload(ss, getFileName(path));
}
```
Here are the functions responsible for those steps.
```C++
/* send request via the socket stream */
void issueRequest(iosockstream& ss, const string& host, const string& path) {
  ss << "GET " << path << " HTTP/1.0\r\n";
  ss << "Host: " << host << "\r\n";
  ss << "\r\n"; /* indicating "the end of my HTTP request" */
  ss.flush();   /* ensure everthing is truely pushed through the web */
}
/* It is not much difference from what we manually type into the terminal
 * when requesting a connection to a server, as shown in 4.1.2.
 * Note that HTTP requires "\r\n" to denote a line-breaking, not "\n" */
```
```C++
/* skip the header of the response, which is written
 * by the server to the socket stream */
void skipHeader(iosockstream& ss) {
  string line;
  do {
    getline(ss, line);
  } while (!line.empty() && line != "\r");
}
```
```C++
const size_t kBufferSize = 1024;
/* save the paload, e.g. texts, images, videos.. */
void savePayload(iosockstream& ss, const string& filename) {
  ofstream output(filename, ios::binary); /* it might not be text */

  /* conutinuously read and write */
  size_t totalBytes = 0;
  while (!ss.fail()) {
    char buffer[kBufferSize] = {'\0'};
    ss.read(buffer, sizeof(buffer));   /* read from stream to buffer */
    totalBytes += ss.gcount();
    output.write(buffer, ss.gcount()); /* write from buffer to file */
  }
  cout << "Total number of bytes fetched: " << totalBytes << endl;
}
```
Note that `getline()` and `read()` will block if there is nothing to read.
### 4.5.3 Some tidbits
- When we create a file to store the payload, we open an `ofstream` object in **binary mode**: `ios::binary`, since the payload might not necessarily be text, and hence we certainly do not want the `ofstream` object to do anything funky with bytes that are accidentally newline characters.
- The HTTP/1.0 protocol dictates that everything beyond that blank line is payload, and that once the server publishes each and every byte of the payload, it closes it's end of the connection. A side's socket closure is the other side's `EOF`.
    > Just like a pipe: write-end's closure is interpreted by the read-end as an `EOF`.
- The HTTP/1.1 protocol allows for connections to remain open  after the initial payload has come through. The code specifically avoids this here by going with HTTP/1.0, which doesn't allow this.

## 4.6 Relevant system calls
### 4.6.1 Need-to-know: IP addresses, domain names
- IP addresses are defined by the [Internet Protocol](https://en.wikipedia.org/wiki/Internet_Protocol) at the Internet layer, and are used to uniquely identify Internet software interfaces. They are not to be confused with MAC addresses, which uniquely identify networking hardware. There are two notable versions of IP: version 4 (IPv4) and version 6 (IPv6).
- IPv4: 32 bits, typically represented by a decimal quad, each component of which is an integer between `0` and `255`. For example: `127.0.0.1`. In implementation, an IPv4 address is packed in a 32-bit integer.
  > SIDE NOTE:<br> There are about 2 billions IPv4 addresses, but now we are running out of them. As one of the early developer of Internet, Stanford University took one class-A IP address range, which contains 2<sup>24</sup> addresses (`36.x.x.x`). In 1993, the IP address distributor convinced Stanford to give it up in exchange for two class-B ranges, which have 2<sup>16</sup>x2 addresses in total.
- IPv6: 128 bits, typically represented by a hexadecimal octet, each component of which is an integer between `0` and `FFFF`, like `20EE:22FF:000A:009A:0000:0000:0000:0000`. Sometimes unnecessary zeros are omitted, as in `20EE:22FF:A:9A::`. In implementation, an IPv6 address is packed in 16 `char`s.
- [Domain Name System](https://en.wikipedia.org/wiki/Domain_Name_System), or DNS, maps an IP address (like a phone number) to a string that is human-friendly (like a person's name), e.g. `54.68.192.252` is mapped with `www.stanford.edu`. Such a string is called a *domain name*. A hierarchical network of DNS server machines handles DNS query.
    > SIDE NOTE:<br>One of the tricks employed by the Great Firewall is [DNS Poisoning](https://en.wikipedia.org/wiki/DNS_spoofing), i.e. altering this mapping relationship on the DNS servers, so they cannot return correct IP addresses of certain websites. Another trick is using HTTP proxies to filter the traffic, which we will see in the assignment.
> SIDE NOTE:<br>You can examine and edit your machine's network interface configuration with command `ifconfig`.<br>You can send ICMP ECHO_REQUEST packets to a host designated by you with command `ping`.

Refer to EE284 or CS144 for more details.

### 4.6.2 Hostname resolution
- Linux uses the `hostent` structure to store a IP address/domain name pair, with other info:
  ```C++
  struct hostent { /* "host entry" */
    char *h_name;       /* host name */
    char **h_aliases;   /* list of host aliases, terminated by NULL */
    int h_addrtype; /* address type. AF_INET for v4, AF_INET6 for v6 */
    int h_length;   /* address length. 4 for v4, 6 for v6 */
    char **h_addr_list; /* list of addresses, terminated by NULL */
  };
  ```
  > "AF" for "address family".
- You can use this system call to acquire the address of the corresponding `hostent` structure from a host name:
  ```C++
  struct hostent *gethostbyname(const char *name); /* get IPv4 addr. */
  struct hostent *gethostbyname2(const char *name);/* get IPv6 addr. */
  ```
- Depending on the type of IP address, an element of `h_addr_list` is a pointer to an `in_addr` or an `in_addr6` structure. We focus on the IPv4 version in our discussion.
  ```C++
  struct in_addr { /* contains one IPv4 address */
    uint32 s_addr; /* big-endian */
  };
  ```
- You can get the IP address string(s) from a `hostent` structure's `h_addr_list` field. In the case of IPv4, you can use this system call to translate an `in_addr` structure to the human-readable address string:
  ```C++
  char *inet_ntoa(struct in_addr in); /* "ntoa": number to ASCII */
  ```
### 4.6.3 The `sockaddr` hierarchy: IP address/port number
```C++
/* all in big-endian */

struct sockaddr_in {           /* IPv4 socket addr./port record */
  unsigned short sin_family;   /* protocol for socket: AF_INET, 2 bytes */
  unsigned short sin_port;     /* port number, 2 bytes */
  struct in_addr sin_addr;     /* IP address, 4 bytes */
  unsigned char sin_zero[8];   /* pad this structure to 16 bytes long */
};

struct sockaddr_in6 {          /* IPv6 socket addr./port record */
  unsigned short sin6_family;  /* protocol for socket: AF_INET6 */
  unsigned short sin6_port;    /* port number */
  /* ... */
};

struct sockaddr {              /* generic socket addr./port record */
  unsigned short sa_family;    /* AF_INET or AF_INET6 */
  char sa_data[14];            /* other data, padding this structure
                                  to 16 bytes long */
};
```
- The `struct sockaddr` type exists as C's best imitation of an abstract base class.
- You rarely declare variables of type `struct sockaddr`, but many system calls will accept parameters of type `struct sockaddr *`, so they can accept either IPv4 or IPv6 versions using one uniform API.
- If a system call accepts a parameter of type `struct sockaddr *`, it really excepts the address of either a `struct sockaddr_in` or a `struct sockaddr_in6`. The system call relies on the value within the first two bytes — the `sa_family` field — to determine what the real type is.

### 4.6.4 Implementing `createClientSocket()`
> You don't need to write this on your own, but since this is not a system call, we want you know it.
```C++
#include ...
using namespace std;

int createClientSocket(const string &host, unsigned short port) {
  /* step 1: get the IP address of the intended host */
  struct hostent *he = gethostbyname(host.c_str());
  if (he == NULL) return -1; /* e.g. the host machine is down */

  /* step 2: get an unused SD */
  int s = socket(AF_INET, SOCK_STREAM, 0);
  if (s < 0) return -1;

  /* step 3: create an (IP, port) pair structure, and fill it with 0's */
  struct sockaddr_in server;
  memset(&server, 0, sizeof(server));

  /* step 4: populate the (IP, port) pair structure */
  server.sin_family = AF_INET;
  /* make unsignshed short big-endian, per networking standard */
  server.sin_port = htons(port);
  server.sin_addr = (in_addr *)((sin_addr *)he->h_addr_list[0]);

  /* step 5: connect the intended (IP, port) */
  if (connect(s, (sockaddr *)&server, sizeof(server)) == 0)
    return s;

  close(s); /* if connect() fails, close s as an FD */
  return -1;
}
```
- System call `socket()`: create a socket. In CS110, we do not have to know the details of it behavior, and always call it this way: `socket(AF_INET, SOCK_STREAM, 0)`. This function gets an unused file descriptor, and configures it as a socket descriptor, and then return it.
  >  `AF_INET`: IPv4 protocol family;<br>`SOCK_STREAM`: if a data package is dropped, then automatically request it again;<br>`0`: choose a specific protocol within the designated protocol family. IPv4 has just one, hence 0.
- System call `connect()` establishes a connection with a server.
  ```C++
  int connect(int sockfd, struct sockaddr *server, int len);
  /* "len" is sizeof(sockaddr) */
  ```
  - It attempts to establish an Internet connection with the server whose IP address and port are specified by `server`, through the socket designated by `sockfd`.
  - **It blocks** until either the connection is successfully established or an error occurs. If successful, the `sockfd` descriptor is ready for reading **and** writing, and the target server has created a designated socket to handle the communication.
    > Note that a socket is bidirectional, but a pipe is unidirectional.
  - It returns `0` on success, `-1` on failure (and `errno` is properly set to indicate the reason).
  - Remember to close `sockfd` if `connect()` fails.

### 4.6.5 Implementing `createServerSocket()`
Listen to a specific port on any of its own IP address.
```C++
#include ...
using namespace std;

static const int kReuseAddresses = 1;   /* 1 means true here */
static const int kDefaultBacklog = 128; /* allow 128 clients to queue */

int createServerSocket(unsigned short port) {
  /* step 1: get an unused SD */
  int s = socket(AF_INET, SOCK_STREAM, 0);
  if (s < 0) return -1;
  /* setsockopt() used here so port becomes available even if
   * server crashes and reboots */
  if (setsockopt(s, SOL_SOCKET, SO_REUSEADDR, 
                 &kReuseAddresses, sizeof(int)) < 0) {
    close(s);
    return -1;
  }

  /* step 2: create an (IP, port) pair structure, and fill it with 0's */
  struct sockaddr_in server;
  memset(&server, 0, sizeof(server));

  /* step 3: populate the (IP, port) pair structure */
  server.sin_family = AF_INET;
  /* make unsignshed short big-endian, per networking standard */
  server.sin_port = htons(port);
  /* make unsignshed long big-endian, per networking standard */
  /* we are creating a listening SD, so it should wait for 
   * requests from any IP address */
  server.sin_addr.s_addr = htonl(INADDR_ANY);

  /* step 4: bind the (IP, port) pair with the socket,
   * and make it a listenning socket */
  if (bind(s, (sockaddr *) &server, sizeof(server)) == 0 
      && listen(s, kDefaultBacklog) == 0) 
    return s;

  close(s);
  return kServerSocketFailure;
}
```
- Note that server socket, or *listening socket*, is **neither** readable **nor** writable. It waits incoming connection request from clients through `accept()` system call, as described in 4.2.5.

### 4.6.6 Summary: steps of creating sockets
#### 4.6.6.1 For client: create a client socket
step 1: get the IP address of the intended server: `gethostbyname()`;

**step 2**: get an unused SD: `socket()`;

**step 3**: create an (IP, port) pair structure of type `sockaddr_in`, and fill it with `0`'s;

**step 4**: populate the (IP, port) pair structure's fields: `.sin_family`, `.sin_port`, `.sin_addr`;
  > Note that `.sin_port`, `.sin_addr` should be the port number and IP address of the other party, i.e. the server, not of the client itself.<br>(remember to make sure these two fields big-endian).

step 5: connect the intended (IP, port) through the socket: `connect()`. If failed, remember to close the socket.

#### 4.6.6.2 For server (1): create a listening socket
**step 1**: get an unused SD: `socket()`;

**step 2**: create an (IP, port) pair structure of type `sockaddr_in`, and fill it with `0`'s;

**step 3**: populate the (IP, port) pair structure's fields: `.sin_family`, `.sin_port`, `.sin_addr`;
  > Note that `.sin_port`, `.sin_addr` should be the port number and IP address of the other party, i.e. the client, not of the server itself. If there is no designated client, pass `INADDR_ANY` as the IP address.<br>(remember to make sure these two fields big-endian).

step 4: bind the (IP, port) pair with the socket: `bind()`, and make it a listening socket: `listen()`. If failed, remember to close the socket.
  > By default, an SD created by `socket()` is not intended for listening (a "passive" role). If you want to make it a *listening socket*, you need to call `listen()` to convert it.

  > Note that `listen()` immediately returns; `accept()` is blocking.

#### 4.6.6.3 For server (2): create a connected socket
Call `accept()` on the server's (only) listening SD. `accept()` blocks the calling thread until a connection request from a client is received, at which point a connected SD will be automatically created and returned by `accept()`. The connected SD should be closed once the client is done.
> Note that `listen()` immediately returns; `accept()` is blocking.
```
a client process (IP, port)                   a server process (IP, port)
┌───────────────────────┐                     ┌───────────────────────┐
│                       │                     []listening SD          │
│                       │                     │                       │
│                       │   another client ◀═▶[]connected SD ◀─▶ R/W  │
│                       │                     :        :              │
│     R/W ◀─▶ client SD[]◀══════socket═══════▶[]connected SD ◀─▶ R/W  │
└───────────────────────┘                     └───────────────────────┘
            A succesfully connected client-server link
```
###### EOF