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
let mut sockets: HashMap<Token, TcpStreak> = HashMap::new();

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
are zero bytes read - this means client has disconnected already, and there is no
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

Writing data to the client socket is similar, and is done via buffer as well, but 
no explicit loop is required, as there is already a method that is doing the 
loop: `.write_all()`.

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

OK, at this point I have reading the data from the socket, and (possible) writing
of the data to the socket. Yet writing never happens, as no token gets registered
for writable events!

OK, when should I register a token for writable event? Well, when it has something
to write! Sounds simple, isn't it? In practice this means it's time to actually 
implement some *protocol*.

### OK, how do I implement a protocol?

TBD

[the-code]: TODO
[mio-github]: https://github.com/tokio-rs/mio
[edge-level]: https://en.wikipedia.org/wiki/Epoll#Triggering_modes
[discard]: https://en.wikipedia.org/wiki/Discard_Protocol
[accept-docs]: https://docs.rs/mio/0.5.1/mio/tcp/struct.TcpListener.html#method.accept
[wouldblock]: https://doc.rust-lang.org/nightly/std/io/enum.ErrorKind.html#variant.WouldBlock
[slab-crate]: https://docs.rs/slab/0.4.2/slab/


