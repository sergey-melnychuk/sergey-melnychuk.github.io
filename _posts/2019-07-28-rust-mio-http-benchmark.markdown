---
layout: post
title:  "Low-level TCP server in Rust with MIO"
date:   2019-07-28 17:22:42 +0200
categories: rust mio
---
It is time to get acquainted with [Metal IO][mio-github], low-level cross-platform 
abstraction over epoll/kqueue written in Rust.

TL;DR: [github][the-code].

In this article I will show and explain how to write simple single-threaded 
asynchronous TCP server, and make it like it can understand HTTP, and then 
benchmark it with `ab`/`wrk`. The results are about to be impressive.

### Getting started

I'm using `mio = "0.6"`.

First, TCP listener is required:

```rust
let address = "0.0.0.0:8080";
let listener = TcpListener::bind(&address.parse().unwrap()).unwrap();
```

Then create `Poll` object and register listener at `Token(0)` for readable events,
activated by edge (not by level). More on [edge vs level][edge-level].

```rust
let poll = Poll::new().unwrap();
poll.register(
    &listener, 
    Token(0),
    Ready::readable(),
    PollOpt::edge()).unwrap();
```

The next essential part is to create `Events` object (of given capacity) and a 
main loop (endless in this case). In the loop the events are polled and processed 
one by one.

```rust
let mut events = Events::with_capacity(1024);
loop {
    poll.poll(&mut events, None).unwrap();
    for event in &events {
        // handle the event
    }
}
```

### Accepting connections (and dropping them)

The event can be one of the following:
* readable event on the listener - means that there are incomint connections that 
are ready to be accepted by the listener
* event on the connected socket
  * readable - the socket has data available for reading
  * writable - the socket is ready for writing some data into it
  
The listener vs socket event can be distinguished by token, where for the listener 
token is always zero, as it was registered in `Poll`.

Below is the simplest event handling approach, accept all incoming connections 
in the loop, and for each connection - simply drop the socket. This will close 
the connection. [Discard Protocol][discard] at your service!

```rust
// handle the event
match event.token() {
    Token(0) => {
        loop {
            match listener.accept() {
                Ok((socket, address)) => {
                    // What to do with the connection?
                    // One option is to simply drop it!
                    println!("Got connection from {}", address);
                },
                Err(ref e) if e.kind() == io::ErrorKind::WouldBlock =>
                    // No more connections ready to be accepted 
                    break,
                Err(e) => 
                    panic!("Unexpected error: {}", e)
            }
        }
    },
    _ => () // Ignore all other tokens
}
```

Listener's `.accept()` returns `std::io::Result<Option<(TcpStream, SocketAddr)>>` 
(as described [here][accept-docs]), and I'm interested in matching only cases when 
non-empty option is returned, or an error. But there is special kind of error 
`io::ErrorKind::WouldBlock` ([doc][wouldblock]), that's basically saying "I'm 
about to wait (block) to make further progress". This is the essential of 
non-blocking behaviour - the point is just not to block (but return respective 
error)! When encountered, such error simply means that there are no more incoming 
connections waiting to be accepted at this point, so the loop is broken, and next 
events get processed.

Now if I run the server and try connecting to it, I can see Discard Protocol in 
action! Amazing right?!

```
$ nc 127.0.0.1 8080
$ 
```

### Registering connections for events

Speaking of next events. In order for next events to occur, they must be registered 
with `Poll` first. Under the hood, `Poll` keeps track which token corresponds to
which socket, but client code only gets access to token. This means if the server 
intends to actually communicate with the client (and I'm pretty sure most servers 
do), then token-socket pair must be stored somehow. In this example, I'm using 
simple `HashMap<Token, TcpStream>`, but [slab][slab-crate] might be more efficient 
to use here.

Token is just a wrapper over `usize`, so simple counter is enough to provide 
increasing sequence of tokens. Once socket is registered with respective token, 
it is inserted into the hash-map.

```rust
let mut counter: usize = 0;
let mut sockets: HashMap<Token, TcpStream> = HashMap::new();

// handle the event
match event.token() {
    Token(0) => {
        loop {
            match listener.accept() {
                Ok((socket, _)) => {
                    counter += 1;
                    let token = Token(counter);

                    // Register for readable events
                    poll.register(&socket, token
                        Ready::readable(),
                        PollOpt::edge()).unwrap();

                    sockets.insert(token, socket);                    
                },
                Err(ref e) if e.kind() == io::ErrorKind::WouldBlock =>
                    // No more connections ready to be accepted 
                    break,
                Err(e) => 
                    panic!("Unexpected error: {}", e)
            }
        }
    },
    token if event.readiness().is_readable() => {
        // Socket associated with token is ready for reading data from it
    }
}
```

### Reading data from client

When readable event occurs for given token, this means data is ready to be read 
from respective socket. I will use just array of bytes as a buffer for reading the
data from the socket. 

Read is performed in the loop until known `WouldBlock` error is returned. Each 
call to read returns (if successful) actual number of bytes read, and when there
are zero bytes read - this [means][read-doc] client has disconnected already, and there is no
point if keeping the socket around (nor continuing the reading loop).

```rust
// Fixed size buffer for reading/writing to/from sockets
let mut buffer = [0 as u8; 1024];
...
token if event.readiness().is_readable() => {
    loop {
        let read = sockets.get_mut(token).unwrap().read(&mut buffer);
        match read {
            Ok(0) => {
                // Successful read of zero bytes means connection is closed
                sockets.remove(token);
                break;
            },
            Ok(len) => {
                // Now do something with &buffer[0..len]
                println!("Read {} bytes for token {}", len, token.0);
            },
            Err(ref e) if e.kind() == io::ErrorKind::WouldBlock => break,
            Err(e) => panic!("Unexpected error: {}", e)
        }
    }
}
...
```

### Writing data to the client

For a token to receive writable event, it must be registered in `Poll` first. The 
`oneshot` option might be useful for scheduling writable events, this option makes 
sure that event of interest is fired only once.

```rust
poll.register(&socket, token
    Ready::writable(),
    PollOpt::edge() | PollOpt::oneshot()).unwrap();
```

Writing data to the client socket is similar, and is done via buffer as well, but 
no explicit loop is required, as there is already a [method][write_all-doc] that 
is doing the loop: `.write_all()`.

```rust
...
token if event.readiness().is_writable() => {
    let socket = sockets.get_mut(&token).unwrap();
    // Now write `len` bytes from buffer to the socket
    socket.write_all(&buffer[0..len]).unwrap();
    // Now what?
}
...
```

### What happens between reading and writing data?

At this point I have reading the data from the socket, and (possible) writing
of the data to the socket. Yet writing never happens, as no token gets registered
for writable events!

When should I register a token for a writable event? Well, when it has something
to write! Sounds simple, isn't it? In practice this means it's time to actually 
implement some *protocol*.

### How do I implement a protocol?

OK, I just want to send text (or JSON) back, and TCP is a [protocol][protocol-doc], 
why do I need more? Well, [TCP][tcp-doc] *is* a protocol, the Transmission Control 
Protocol, the transport-level one. TCP cares about receiver to receive exact
amount of bytes in exact order that sender sent! So at the transport level, I have
to deal with two streams of bytes: one going from client to the server, and another
one going straight back.

What's useful when dealing with servers, is an application level protocol (say, HTTP).
The application level protocol can define such entities as `request`, that server 
receives from the client, and `response`, that client receives back from the server.

It is important to mention, that implementing HTTP *correctly* is not as easy as 
it sounds, and even more, it is already done for you, e.g. in [hyper][hyper-github]. 
Here I won't bother with actually implementing HTTP, what I'm going to do instead 
is to make my server behave as if it really understands GET requests, but will 
always respond to such request with a response containing 6 bytes: `b"hello\n"`.

If I want all my protocol to do is to return how many bytes were received, I will 
need an actual number of bytes written (`HashMap` will do), count bytes when 
readable event occurs, then schedule a one-shot writable event, and when writable
event occurs - send the response and drop the connection.

```rust
let mut response: HashMap<Token, usize> = HashMap::new();
...
token if event.readiness().is_readable() => {
    let mut bytes_read: usize = 0;
    loop {
        ... // sum up number of bytes received
    }
    response.insert(token, bytes_read);
    // re-register for oneshot writable event
}
...
token if event.readiness().is_writable() => {
    let n_bytes = response[&token];
    let message = format!("Received {} bytes\n", n_bytes);
    sockets.get_mut(&token).unwrap().write_all(message.as_bytes()).unwrap();
    response.remove(&token);
    sockets.remove(&token); // Drop the connection
},
```

### Mocking HTTP

For the sake of this article, mocking of HTTP is more than enough. I will use the 
fact that HTTP request header is separated from request body (if any) with 4 bytes, 
`\r\n\r\n`. So if I keep track of what current client have sent, and if at any point 
there are target 4 bytes in there, I can respond with pre-defined HTTP response:

```
HTTP/1.1 200 OK
Content-Type: text/html
Connection: keep-alive
Content-Length: 6

hello
```

Simple `HashMap` is enough to keep track of all the bytes received:

```rust
let mut requests: HashMap<Token, Vec<u8>> = HashMap::new();
```

Once reading is done, it makes sense to check if request is ready:

```rust
fn is_double_crnl(window: &[u8]) -> bool {  /* trivial */ }

let ready = requests.get(&token).unwrap()
    .windows(4)
    .find(|window| is_double_crnl(*window))
    .is_some();
```

And if it is, it's time to schedule some writing!

```rust
if ready {
    let socket = sockets.get(&token).unwrap();
    poll.reregister(
        socket,
        token,
        Ready::writable(),
        PollOpt::edge() | PollOpt::oneshot()).unwrap();
}
```

After writing is completed, it is important to keep the connection open, and 
`reregister` socket for reading again.
 
```rust
poll.reregister(
    sockets.get(&token).unwrap(),
    token,
    Ready::readable(),
    PollOpt::edge()).unwrap();
```

Server is ready!

```
$ curl localhost:8080
hello
```

So the fun can start - let's try an see how performant this *single-threaded* 
server is with common tools: `ab` and `wrk`. Note: 
* `ab` requires `-k` option to use `keep-alive` and reuse existing connections
* `wrk2` is actually used as `wrk`, thus `--rate` parameter is present
* `ab`/`wrk` is running on different VM than the server (but in the same region)

Here are the numbers I got when trying benchmarking on the instance 
`n1-standard-8 (8 vCPUs, 30 GB memory)` of some cloud provider 
(that I'm not really allowed to mention here).

```
$ ab -n 1000000 -c 128 -k http://instance-1:8080/
<snip>
Requests per second:    105838.76 [#/sec] (mean)
Transfer rate:          9095.52 [Kbytes/sec] received
```

```
$ wrk -d 60s -t 8 -c 128 --rate 150k http://instance-1:8080/
<snip>
Requests/sec: 120596.75
Transfer/sec: 10.12MB
```

105k and 120k rps, not bad for a single thread.

Of course, it is cheating, and one thing that's benchmarked is actually throughput
between user and kernel space, since no single byte actually hits the network. But 
still, this can be a (more or less) meaningful bottom line for how fast networking
can be done using only one thread.

The full and runnable code is available on [github][the-code], organized by one
logical chapter per pull-request:
* init project: [PR#1](https://github.com/sergey-melnychuk/mio-tcp-server/pull/1)
* accept & discard: [PR#2](https://github.com/sergey-melnychuk/mio-tcp-server/pull/2)
* read from socket: [PR#3](https://github.com/sergey-melnychuk/mio-tcp-server/pull/3)
* writing to socket: [PR#4](https://github.com/sergey-melnychuk/mio-tcp-server/pull/4)
* mocking HTTP: [PR#5](https://github.com/sergey-melnychuk/mio-tcp-server/pull/5)

### Where to go from here

Scaling to multiple threads: start [here](https://blog.cloudflare.com/the-sad-state-of-linux-socket-balancing/).

[the-code]: https://github.com/sergey-melnychuk/mio-tcp-server
[mio-github]: https://github.com/tokio-rs/mio
[edge-level]: https://en.wikipedia.org/wiki/Epoll#Triggering_modes
[discard]: https://en.wikipedia.org/wiki/Discard_Protocol
[accept-docs]: https://docs.rs/mio/0.5.1/mio/tcp/struct.TcpListener.html#method.accept
[wouldblock]: https://doc.rust-lang.org/nightly/std/io/enum.ErrorKind.html#variant.WouldBlock
[slab-crate]: https://docs.rs/slab/0.4.2/slab/
[read-doc]: https://doc.rust-lang.org/nightly/std/io/trait.Read.html#tymethod.read
[write_all-doc]: https://doc.rust-lang.org/std/io/trait.Write.html#method.write_all
[tcp-doc]: https://ru.wikipedia.org/wiki/Transmission_Control_Protocol
[protocol-doc]: https://en.wikipedia.org/wiki/Communication_protocol
[hyper-github]: https://github.com/hyperium/hyper
