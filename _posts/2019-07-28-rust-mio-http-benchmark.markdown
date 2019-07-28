---
layout: post
title:  "Low-level TCP server in Rust with MIO"
date:   2019-07-28 17:22:42 +0200
categories: blog jekyll github
---
It is time to get acquainted with [Metal IO][mio-github], low-level cross-platform abstraction over epoll/kqueue written in Rust.

In this article I will show and explain how to write simple single-threaded TCP server, implement protocol that mocks HTTP and benchmark it with `ab` and `wrk`.

TDB

[mio-github]: https://github.com/tokio-rs/mio

