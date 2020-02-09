---
layout: post
title:  "Multi-threaded TCP server in Rust with MIO"
date:   2020-02-09 22:14:42 +0200
categories: rust mio concurrency multi-threading
---
Building up on my previous posts about MIO-based [server][post-server] and [parser combinators][post-parsers], this post is about making a simple TCP server run on multiple threads and implement WebSocket protocol.

TL;DR: [code].

[post-server]: https://sergey-melnychuk.github.io/2019/08/01/rust-mio-tcp-server/
[post-parsers]: https://sergey-melnychuk.github.io/2019/08/31/rust-parser-combinators/
[code]: https://github.com/sergey-melnychuk/mio-websocket-server
