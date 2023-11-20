---
layout: post
title:  "Get rid of typos with typos"
date:   2023-11-20 09:55:42 +0200
categories: rust cargo clippy typos
---

Typos in code and comments can be annoying. But they don't have to be, with [`typos`](https://github.com/crate-ci/typos) it is super easy to introduce automated typos checking (and fixing).

Easy steps to get rid of typos. Once and for all.

1. Install `typos` with `cargo install typos-cli` or [any other way](https://github.com/crate-ci/typos#install).
1. Run `typos` to detect.
1. Run `typos -w` to fix.
1. ???
1. PROFIT!

Maybe add a [Github Action](https://github.com/crate-ci/typos/blob/master/docs/github-action.md):

```
name: ...
on: [pull_request]

jobs:
  ...
  typos:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: crate-ci/typos@v1.16.23
        with:
          files: .
```

That's all folks!
