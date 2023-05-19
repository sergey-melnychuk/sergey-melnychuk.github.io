---
layout: post
title:  "Cloudflare Workers setup and usage"
date:   2023-05-19 15:11:42 +0200
categories: cloudflare workers wasm rust
---

TL;DR: See [sergey-melnychuk/project-one](https://github.com/sergey-melnychuk/project-one/).

At the time of writing this post [Cloudflare Workers](https://developers.cloudflare.com/workers/) says:

> Deploy serverless code instantly across the globe to give it exceptional performance, reliability, and scale.

Basically looks like an AWS Lambda but with benfits of WASM runtime (scalability, performance, no "cold starts", no network overhead).

## Setup

Install [wrangler](https://developers.cloudflare.com/workers/wrangler/install-and-update/) and follow [Project One](https://github.com/sergey-melnychuk/project-one/) (note that dependency versions are likely to be outdated):

#### (Cargo.toml)

```
[package]
name = "project-one"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
worker = "0.0.11"
```

#### (wrangler.toml)

```
name = "project-one"
compatibility_date = "2022-10-10"

main = "build/worker/shim.mjs"

[build]
command = "worker-build --release"
```

#### (src/lib.rs)

```
use worker::*;

#[event(fetch)]
pub async fn main(_req: Request, _env: Env, _ctx: Context) -> Result<Response> {
    // TODO: do your thing
    Response::empty()
}
```

#### Build

```
cargo install worker-build
worker-build --release
```

#### Run locally (development mode)

```
$ wrangler dev
 ⛅️ wrangler 3.0.0
------------------
wrangler dev now uses local mode by default, powered by 🔥 Miniflare and 👷 workerd.
To run an edge preview session for your Worker, use wrangler dev --remote
Running custom build: worker-build --release
[INFO]: 🎯  Checking for the Wasm target...
[INFO]: 🌀  Compiling to Wasm...
    Finished release [optimized] target(s) in 0.06s
[INFO]: ⬇️  Installing wasm-bindgen...
[INFO]: Optimizing wasm binaries with `wasm-opt`...
[INFO]: Optional fields missing from Cargo.toml: 'description', 'repository', and 'license'. These are not necessary, but recommended
[INFO]: ✨   Done in 2.43s
[INFO]: 📦   Your wasm pkg is ready to publish at /home/work/Temp/project-one/build.

  shim.mjs  11.8kb

⚡ Done in 5ms
⎔ Starting local server...
[mf:inf] Ready on http://127.0.0.1:8787/
<snip>
```

NOTE: Running on Ubuntu 22.04 needs `sudo apt install libc++1`. More details [here](https://github.com/cloudflare/workers-sdk/issues/3262).

#### Deploy to Cloudflare

```
# export CLOUDFLARE_API_TOKEN=<snip>
# export CLOUDFLARE_ACCOUNT_ID=<snip>
wrangler publish
```

After publishing, you get a public URL to trigger the worker.
