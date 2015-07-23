---
layout: page
title: "About"
permalink: /about/
---

Experienced software engineer passionate about functional programming, large-scale distributed systems and machine learning. Constantly pursuing self-improvement both personal and professional. With total of 15+ (getting paid for programming since 2008) years in the industry, I am proficient mostly in Rust (3+ years of day-to-day experience), Java (10+ years) and Scala (5+ years) but also have meaningful experience in Go, C++, C, Python and JavaScript.

Strong background in Computer Science and knowledge of advanced data structures & algorithms (with Master's Degree in Computer Science since 2012), concurrent & parallel programming, distributed systems and algorithms, as well as TRIZ and Design Thinking allows designing and building clean, readable, testable, scalable software solutions for sophisticated problems. I don't stick to single language/framework/stack and keep a generalist approach that allows picking the right tool for a job, taking into account requirements and trade-offs.

Professional experience with **Rust**:

- [Eiger](https://www.eiger.co) (1+ year):
  - development and maintenance and of a Starknet full node: [Pathfinder](https://github.com/eqlabs/pathfinder)
  - JSON-RPC code generation tool: [iamgroot](https://github.com/sergey-melnychuk/iamgroot)
  - PoC of a Starknet full node: [Armada](https://github.com/sergey-melnychuk/armada)

- NEAR (via [GitCoin](https://www.gitcoin.co/))
  - AVL-tree implementation for [Near Rust SDK](https://docs.near.org/sdk/rust/introduction)
    - [Pull Request](https://github.com/near/near-sdk-rs/pull/154)
  - Heap-based Priority Queue (not merged though)
    - [Pull Request](https://github.com/near/near-sdk-rs/pull/170)

Things I did in **Rust**:

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

- [Advent of Code](https://adventofcode.com) solutions:
  - [2022](https://github.com/sergey-melnychuk/advent-of-code-2022)
  - [2021](https://github.com/sergey-melnychuk/advent-of-code-2021)
  - [2020](https://github.com/sergey-melnychuk/advent-of-code-2020)
