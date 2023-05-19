---
layout: post
title:  "AWS Lambda memory usage"
date:   2023-05-12 15:12:42 +0200
categories: aws lambda memory profiling rust
---

AWS Lambda shows memory peak-usage, but when memory usage hits the limit, breaking down where exactly too much memory is being used might be a very useful things. The runtime is kind of a black box (at least in my understanding), so any telemetry/measurements need to be baked into the function. For this example, I only describe an approach useful for Rust runtime.

## Setup

Just use [cargo lambda](https://www.cargo-lambda.info/guide/getting-started.html):

```
## build from source and install using cargo
cargo install --locked cargo-lambda

## create project
cargo lambda new new-lambda-project && cd new-lambda-project
```

#### new-lambda-project/Cargo.toml

```
[package]
name = "new-lambda-project"
version = "0.1.0"
edition = "2021"

# Starting in Rust 1.62 you can use `cargo add` to add dependencies
# to your project.
#
# If you're using an older Rust version,
# download cargo-edit(https://github.com/killercup/cargo-edit#installation)
# to install the `add` subcommand.
#
# Running `cargo add DEPENDENCY_NAME` will
# add the latest version of a dependency to the list,
# and it will keep the alphabetic ordering for you.

[dependencies]
lambda_http = { version = "0.8.0", default-features = false, features = ["apigw_http"] }
lambda_runtime = "0.8.0"
tokio = { version = "1", features = ["macros"] }
tracing = { version = "0.1", features = ["log"] }
tracing-subscriber = { version = "0.3", default-features = false, features = ["fmt"] }
```

#### new-lambda-project/src/main.rs

```rust
use lambda_http::{run, service_fn, Body, Error, Request, RequestExt, Response};

/// This is the main body for the function.
/// Write your code inside it.
/// There are some code example in the following URLs:
/// - https://github.com/awslabs/aws-lambda-rust-runtime/tree/main/examples
async fn function_handler(event: Request) -> Result<Response<Body>, Error> {
    // Extract some useful information from the request
    let who = event
        .query_string_parameters_ref()
        .and_then(|params| params.first("name"))
        .unwrap_or("world");
    let message = format!("Hello {who}, this is an AWS Lambda HTTP request");

    // Return something that implements IntoResponse.
    // It will be serialized to the right response event automatically by the runtime
    let resp = Response::builder()
        .status(200)
        .header("content-type", "text/html")
        .body(message.into())
        .map_err(Box::new)?;
    Ok(resp)
}

#[tokio::main]
async fn main() -> Result<(), Error> {
    tracing_subscriber::fmt()
        .with_max_level(tracing::Level::INFO)
        // disable printing the name of the module in every log line.
        .with_target(false)
        // disabling time is handy because CloudWatch will add the ingestion time.
        .without_time()
        .init();
```

## Memory Usage

Slightly modified version of an approach using [jemalloc](https://docs.rs/jemalloc-ctl/latest/jemalloc_ctl/index.html) described on [Stack Overflow](https://stackoverflow.com/a/30983834).

Add necessary dependencies:

```
[dependencies]
...
jemallocator = "0.5"
jemalloc-sys = {version = "0.5", features = ["stats"]}
libc = "0.2"
jemalloc-ctl = "0.5.0"
```

Add global allocator definition and use snapshot memory usage before/after memory-intense operations:

```
#[global_allocator]
static ALLOC: jemallocator::Jemalloc = jemallocator::Jemalloc;

fn memory(tag: &str) {
    use jemalloc_ctl::{epoch, stats};
    epoch::advance().unwrap();

    let active = stats::active::read().unwrap() >> 20 + 1;
    let allocated = stats::allocated::read().unwrap() >> 20 + 1;
    let resident = stats::resident::read().unwrap() >> 20 + 1;
    let metadata = stats::metadata::read().unwrap() >> 20 + 1;
    tracing::warn!("MEMORY '{}': active: {} Mb, allocated: {} Mb, resident: {} Mb, metadata: {} Mb", 
        tag, active, allocated, resident, metadata);
}

async fn do_stuff() -> Result<(), Error> {
    memory("before");

    // do this

    memory("this");

    // do that

    memory("that");
}
```

## Next Steps

With memory stats dumped and memory "hot spots" detected, they can be addressed:

- [Rest Perf book](https://nnethercote.github.io/perf-book/title-page.html)
- [Valgrind DHAT](https://valgrind.org/docs/manual/dh-manual.html)
- [DHAT crate](https://docs.rs/dhat/latest/dhat/)
- ["Measuring Memory Usage in Rust"](https://rust-analyzer.github.io/blog/2020/12/04/measuring-memory-usage-in-rust.html)
- ["Measuring data structure sizes"](https://blog.mozilla.org/nnethercote/2015/06/03/measuring-data-structure-sizes-firefox-c-vs-servo-rust/)
