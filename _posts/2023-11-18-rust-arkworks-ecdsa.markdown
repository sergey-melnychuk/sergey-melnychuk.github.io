---
layout: post
title:  "Learning ECDSA with arkworks-rs"
date:   2023-11-18 14:16:42 +0200
categories: rust crypto arkworks ecdsa
---
While I was [doing-some-blockchain](https://github.com/sergey-melnychuk/doing-some-blockchain) recently, I have found out that Elliptic Curve algebra does not seem to fit easily into `u128`, so this is a second attepmt, now done properly, using [arkworks-rs](https://github.com/arkworks-rs).



References:
- [Getting started with arkworks-rs](https://medium.com/@prajwolgyawali/getting-started-with-arkworks-rs-e5ceaca895a9)
- [Applied Elliptic Curve Cryptography](https://ventral.digital/posts/2023/8/22/applied-elliptic-curve-cryptography)
