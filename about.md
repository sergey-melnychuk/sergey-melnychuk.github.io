---
layout: page
title: "About"
permalink: /about/
---

Experienced software engineer passionate about functional programming, distributed systems and machine learning. Constantly pursuing self-improvement both personal and professional.

Code:

- AVL-tree implementation (included in Near Rust SDK)
  - provides API similart to [std::collections::BTreeMap](https://doc.rust-lang.org/std/collections/struct.BTreeMap.html)
  - [Pull Request](https://github.com/near/near-sdk-rs/pull/154)

- YAKVDB: B-tree based persistent key-value datastore
  - supports insert/lookup/remove (obviously)
  - supports ascending/descending traversals
  - uses LRU page cache to limit memory footprint
  - [code](https://github.com/sergey-melnychuk/yakvdb)

- Uppercut: Opinionated "pure" Actor Model implementation
  - based on the original paper by Carl Hewitt
  - supports message passing over the network
  - examples contain implementations of Gossip and PAXOS protocols
  - [code](https://github.com/sergey-melnychuk/uppercut)

- Basic web server based on uppercut (outperforms actix-web on very specific setup)
  - Uses actor-per-connection model and MIO
  - [code](https://github.com/sergey-melnychuk/uppercut-lab/tree/master/uppercut-mio-server)

- Parsed: Monadic parser combinators library
  - includes parsers for HTTP request and WebSocket frame
  - [code](https://github.com/sergey-melnychuk/parsed)
  - [blog](https://sergey-melnychuk.github.io/2019/08/31/rust-parser-combinators/)

- Very basic HTTP and WebSocket server implementation
  - Multi-threaded
  - Based on MIO
  - [code](https://github.com/sergey-melnychuk/mio-websocket-server)
  - [blog](https://sergey-melnychuk.github.io/2020/04/27/multi-threaded-http-websocket-server-in-rust/)

- YES-lang: Interpreter of a simple programming language
  - supports functions as "first-class citizens"
  - closures support
  - Pratt parser implementation
  - [code](https://github.com/sergey-melnychuk/yes-lang)

- My "two cents" in rust-lang:
  - [Pull Request](https://github.com/rust-lang/rust/pull/72206)

- Advent of Code (fully solved ones)
  - [2021](https://github.com/sergey-melnychuk/advent-of-code-2021)
  - [2020](https://github.com/sergey-melnychuk/advent-of-code-2020)
