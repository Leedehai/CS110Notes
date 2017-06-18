# Topic 4 Networking (3)
<div id="author-signature">Leedehai</div>
Friday, May 26, 2017

## 4.7 A networking API
Let's say you have an exectable called `srabble-word-finder`, which takes a string as input and outputs a list of strings. We want to write a program that serves as the API of this executable so we can feed inputs and send outputs via networks.
```C++
#include ...

using namespace std;

struct subprocess_t {
  pid_t pid;
  int supplyfd;
  int ingestfd;
};

subprocess_t subprocess(char *argv[], 
                        bool supplyChildInput, 
                        bool ingestChildOutput) { /* in assignment 2 */ }

static void publishScrabbleWords(int clientSocket) {
  /* get input from network */
  sockbuf sb(clientSocket);
  iosockstream ss(&sb);
  string letters = getLetters(ss);
  skipHeaders(ss);

  /* spawn a child process to handle it - remember to rip the zombie */
  const char *command[] = {"./scrabble-word-finder", 
                           letters.c_str(), NULL};
  subprocess_t sp = subprocess(const_cast<char **>(command), false, true);
  vector<string> formableWords = pullFormableWords(sp.ingestfd);
  waitpid(sp.pid, NULL, 0);
  
  /* send the output to network */
  ostringstream payload;
  constructPayload(formableWords, payload);
  sendResponse(ss, payload.str());
}
```
An illustration:
```
                 network   
                    ▲
                    ║
                  socket
                    ║
- - - -┌────────────▼─────────────┐- - - -
       │ API: publishSrabbleWords │  (hidden from the network's
       │    calls subprocess()    │   point of view)
       └──────┬───────────▲───────┘
             pipe        pipe       
       ┌──────▼───────────┴───────┐
       │      child process:      │
       │  ./scrabble-word-finder  │
       └──────────────────────────┘
```

## 4.8 A bite of MapReduce
> *MapReduce* is a programming model, as well as an associated implementation of that model, for processing and generating large data sets with a **parallel, distributed algorithm** on a cluster. The model is an specialization of the *split-apply-combine* strategy for data analysis.

A silly example: counting words. Say, we have a document and we would like to count the absolute frequency of these words, and we have multiple machines at our disposal:
```
a b c d a b c
```
In this case, we want the end result to look like this:
```
a:2 b:2 c:2 d:1
```
The MapReduce pipeline looks like this:
```
("┆" separates different machines)

                             [input file(s)]  a  b  c  d  a  b  c

(1) Mapping by mappers on 2 machines                   │
    (forming trivial key-value pairs)                  ▼

                 [intermediate file(s)]  a:1 b:1 c:1 d:1 a:1 ┆ b:1 c:1
                                                (m1.txt)       (m2.txt)

(2) Assigned to reducers on 3 machines                 │
    (a,b->1, c->2, d->3, and reducers sort             ▼
      what they receive)
                                         a:1 a:1 b:1 b:1 ┆ c:1 c:1 ┆ d:1 

(3) Grouping by keys by reducers on 3 machines         │
    (the end result is still sorted)                   ▼

                                      a:(1,1) b:(1,1) ┆ c:(1,1) ┆ d:(1)

(4) Reducing by reducers on 3 machines                 │
                                                       ▼

                         [output file(s)]  a:2 b:2 ┆  c:2   ┆   d:1
                                           (r1.txt) (r2.txt) (r3.txt)
```
- Our last assignment will be about using MapReduce to count the words in Homer's *The Odyssey*, utilizing the 32 machines in our `myth` cluster, as the capstone of our odyssey of CS110 :)

Refer to CS246 for more details.
###### EOF