---
layout: post
title:  "Multi-threaded TCP server in Rust with MIO"
date:   2020-02-09 22:14:42 +0200
categories: rust mio concurrency multi-threading
---
Building up on my previous posts about MIO-based [server][post-server] and [parser combinators][post-parsers], this post is about making a simple TCP server run on multiple threads and implement WebSocket protocol.

TL;DR: [code].

Running `wrk` on `8 vCPUs, 30 GB` machine:

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

[post-server]: https://sergey-melnychuk.github.io/2019/08/01/rust-mio-tcp-server/
[post-parsers]: https://sergey-melnychuk.github.io/2019/08/31/rust-parser-combinators/
[code]: https://github.com/sergey-melnychuk/mio-websocket-server
