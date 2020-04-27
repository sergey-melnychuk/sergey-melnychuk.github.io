---
layout: post
title:  "Multi-threaded HTTP/WebSocket server in Rust"
date:   2020-04-27 21:15:42 +0200
categories: rust mio concurrency multi-threading tcp websocket
---
Building up on my previous posts about [MIO-based server][post-server] and 
[parser combinators][post-parsers], this post is about making a very simple HTTP 
server capable of running on multiple threads and implementing WebSocket protocol.

TL;DR: [code].

### Benchmark

Quick benchmark with `wrk` on `8 vCPUs, 30 GB` machine shows 110k rps vs 280 rps 
when distributing socket reading/writing over 8 threads. **Important note**: this 
benchmark is not representative on its own, just the comparison of two allows to 
notice 2.5x speedup. [Amdahl's Law](https://en.wikipedia.org/wiki/Amdahl%27s_law) 
in action: the main thread is still responsible for listening for incoming 
connections and registering socket events.

#### Single-threaded

```
instance-1:~/mio-tcp-server$ wrk -d 1m -c 128 -t 8 http://127.0.0.1:8080/
Running 1m test @ http://127.0.0.1:8080/
  8 threads and 128 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.15ms  131.93us   2.65ms   68.80%
    Req/Sec    13.91k     0.86k   19.76k    66.96%
  6645523 requests in 1.00m, 557.71MB read
Requests/sec: 110731.94
Transfer/sec:      9.29MB
```

#### Multi-threaded (8 cores, 8 threads)

```
instance-1:~/mio-websocket-server$ wrk -d 1m -c 128 -t 8 http://127.0.0.1:9000/
Running 1m test @ http://127.0.0.1:9000/
  8 threads and 128 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   479.65us  554.02us  28.64ms   96.33%
    Req/Sec    35.09k     2.01k   59.50k    72.71%
  16765024 requests in 1.00m, 1.37GB read
Requests/sec: 279225.41
Transfer/sec:     23.43MB
```

### Key Concepts

#### Handler

A container for token, actual socket and send/receive buffers. A `ByteStream` from 
[parser-combinators](https://sergey-melnychuk.github.io/2019/08/31/rust-parser-combinators/)
is useful for stateless parsing (receive stream) and buffered sending (send stream).

Handle wraps a socket provided from listener as a connection, and has `pull()` to read from
socket into receive stream, `push()` to write data from send stream to the socket, and `put()`
to store data for buffering into the send stream.

```rust
struct Handler {
    token: Token,
    socket: TcpStream,
    is_open: bool,
    recv_stream: ByteStream,
    send_stream: ByteStream,
}
```

#### Worker Thread

Worker Thread receives events (handlers) from the main thread and runs the "payload".
Then returns handler (most likely in updated state) back to the main thread.

```rust
loop {
    let mut handler = event_rx.lock().unwrap().recv().unwrap();
    debug!("token {} background thread", handler.token.0);
    handler.pull();

    // do something useful here

    handler.push();
    ready_tx.send(handler).unwrap();
}
```

The payload ("something useful" part) might be actually parsing the receive buffer:

```rust
fn handle(req: Request) -> Response { ... }

if let Some(req) = parse_http_request(&mut handler.recv_stream) {
    handler.recv_stream.pull(); // roll over the receive stream
    let res = handle(req);      // handle the request - get response
    handler.put(res);           // put response into send stream
}
```

#### Listener Thread

The Main Thread owns server socket that receives connections and owns `Poll` that
allows getting and processing socket events. Once read/write event for specific handler was
received, it is time to send the handler to worker thread for processing.

Meanwhile, handlers that are returning from workher threads need re-registering for next
socket events. Thus logical second duty of a Listener Thread is to re-register handlers for 
respective socket events: if a handler has non-empty send stream, it needs to receive writable
event - if not the assumption is that it is ready to read some more data.

```rust
loop {
    poll.poll(&mut events, Some(Duration::from_millis(20))).unwrap();
    // 1. process socket events
    for event in &events {
        match event.token() {
            Token(0) => {
                loop {
                    match listener.accept() {
                        Ok((socket, _)) => {
                            // accept connection, create Handler
                        },
                        Err(_) => break
                    }
                }
            },
            token if event.readiness().is_readable() => {
                debug!("token {} readable", token.0);
                if let Some(handler) = handlers.remove(&token) {
                    event_tx.send(handler).unwrap();
                }
            },
            token if event.readiness().is_writable() => {
                debug!("token {} writable", token.0);
                if let Some(handler) = handlers.remove(&token) {
                    event_tx.send(handler).unwrap();
                }
            },
            _ => unreachable!()
        }
    }
    
    // 2. process updates received from handlers
    loop {
        let opt = ready_rx.try_recv();
        match opt {
            Ok(handler) if !handler.is_open => {
                // socket is closed, drop the handler
            },
            Ok(handler) => {
                if handler.send_stream.len() > 0 {
                    // register handler for writing
                } else {
                    // register handler for reading
                }
                handlers.insert(handler.token, handler);
            },
            _ => break,
        }
    }
}
```

#### HTTP to WebSocket

WebSocket Upgrade request is just a regular HTTP request, but it needs some special handling: like
calculating 'Sec-Websocket-Accept' response header based on 'Sec-Websocket-Key' request header.

```rust
fn res_sec_websocket_accept(req_sec_websocket_key: &String) -> String {
    let mut hasher = Sha1::new();
    hasher.input(req_sec_websocket_key.to_owned() + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11");
    base64::encode(hasher.result())
}
```

For more details on WebSocket: see nice guide on [MDN](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API/Writing_WebSocket_servers)
and rust-parser-combinators [code](https://github.com/sergey-melnychuk/rust-parser-combinators/blob/master/src/ws.rs). I must admin
parsing binary WebSocket frames is as straightforward as parsing text-based HTTP requests.

#### It Works!

Finally, putting all pieces together allows to connect to the server from a browser:

![WebSocket connection in Browser]({{ site.url }}/assets/2020-04-27-multi-threaded-http-websocket-server-in-rust/websocket.png)

#### Plans

1. The more I'm moving towards fundamental constructs, them more it seems like Actor Model. 
So I have already been [doing-some-actors](https://github.com/sergey-melnychuk/doing-some-actors) for some time.
1. With clean and simple Actor Model implementation and HTTP/WebSocket protocol parser, the classic demo
would be to build... a chat application! This is what is coming next, most likely.
1. Actor from the Actor Model seems to be way too low-level for direct usage in applications. 
   - Somehow many people don't feel wrong writing `class User extend Actor` (thus coupling domain-model entity with 
specific Actor Model implementation) - for me it seems the same as writing `class User extends Mutex`...
   - Thus nice and clean (and preferably type-safe) API on top of that might be useful!

[post-server]: https://sergey-melnychuk.github.io/2019/08/01/rust-mio-tcp-server/
[post-parsers]: https://sergey-melnychuk.github.io/2019/08/31/rust-parser-combinators/
[code]: https://github.com/sergey-melnychuk/mio-websocket-server
