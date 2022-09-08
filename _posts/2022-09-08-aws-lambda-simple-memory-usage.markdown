---
layout: post
title:  "Simple memory usage stats for AWS Lambda"
date:   2022-09-08 12:48:42 +0200
categories: aws lambda memory profiling
---

TODO:
- cargo lambda project setup and deployment
- running/debugging locally
- collecting simple memory stats with jemalloc
- further reading (DHAT, valgrind mastif)

[cargo-lambda]: https://www.cargo-lambda.info/
[rust-perf-book]: https://nnethercote.github.io/perf-book/title-page.html
[measuring-memory]: https://rust-analyzer.github.io/blog/2020/12/04/measuring-memory-usage-in-rust.html
[dhat-crate]: https://docs.rs/dhat/latest/dhat/
[stackoverflow-answer]: https://stackoverflow.com/a/30983834
[jemalloc-ctl-docs]: https://docs.rs/jemalloc-ctl/latest/jemalloc_ctl/index.html
[measure-data-sizes]: https://blog.mozilla.org/nnethercote/2015/06/03/measuring-data-structure-sizes-firefox-c-vs-servo-rust/
