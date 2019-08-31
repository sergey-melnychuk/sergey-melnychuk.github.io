---
layout: post
title:  "Monadic parser combinators in Rust"
date:   2019-08-31 18:24:42 +0200
categories: parser combinators monad rust
---
Why don't I implement a nice monadic parser combinator library in Rust? That's what my thought was when after implementing low-level mock-HTTP server in [MIO][mio] and actually needed to parse the bytes received by server.

What I wanted is a declative way to define sequence of strings to be matched and/or extracted. Example of a parser for a string representing key-value pair (`<string>: <string>`) would look like this:

```rust
#[derive(Default)]
struct KV {
    k: String,
    v: String,
}

let p: Parser<KV> = Parser::init(Entity::default())
    .then(until(':'))
    .save(|target, key| target.k = key)
    .then(exact(": "))
    .save(|target, val| target.v = val);

let stream = "env: prod".to_stream();
let kv: KV = p.parse(stream).unwrap();
// returns KV { k: "env", v: "prod" }
```

The re-invented stream abstraction is useful when it is required to pull bytes or chunks of bytes from a buffer, with a possibility to reset the position some bytes are read (this allows re-trying matching if needed).

The implementation has a few key points. The `Matcher` type - just a function that can be applied to a `ByteStream`. `ByteStream` itself is just a wrapper over `Vec<u8>`, nothing fancy.

```rust
pub type Matcher<T> = dyn Fn(&mut ByteStream) -> Result<T, MatchError> + 'static;
```

The `single` matcher that matches a single character from input `ByteStream` looks like this:

```rust
pub fn single(chr: char) -> Box<Matcher<char>> {
    Box::new(move |bs| {
        let pos = bs.pos();
        bs.next()
            .map(|b| b as char)
            .filter(|c| *c == chr)
            .ok_or(MatchError::unexpected(
                pos,
                format!("EOF"),
                format!("{}", chr),
            ))
    })
}
```

The `repeat` matcher is a more generic one, that allows matching zero or more occurrences of an input matcher:

```rust
pub fn repeat<T: 'static>(this: Box<Matcher<T>>) -> Box<Matcher<Vec<T>>> {
    Box::new(move |bs| {
        let mut acc: Vec<T> = vec![];
        loop {
            let mark = bs.mark();
            match (*this)(bs) {
                Err(_) => {
                    bs.reset(mark);
                    return Ok(acc);
                }
                Ok(t) => acc.push(t),
            }
        }
    })
}
```

There are more useful matchers: `one`, `maybe`, `until`, `before`, `token`, `string`, `space` and `bytes`. I believe it is possible to infer underlying functionality from the name of a matcher.

To chain two matchers into a new matcher, that represents sequential match made by the first one and then by the second one, there is a `chain` operator:

```rust
// Given Matcher<T> and Matcher<U>, make a Matcher<(T, U)>.
pub fn chain<T: 'static, U: 'static>(
    this: Box<Matcher<T>>,
    next: Box<Matcher<U>>,
) -> Box<Matcher<(T, U)>> {
    Box::new(move |bs| {
        let t = (*this)(bs)?;
        let u = (*next)(bs)?;
        Ok((t, u))
    })
}
```

To manipulate the matched content, there is a `map` operator:

```rust
// Just apply 'f' on result of a match
pub fn map<T: 'static, U: 'static, F: Fn(T) -> U + 'static>(
    this: Box<Matcher<T>>,
    f: F,
) -> Box<Matcher<U>> {
    Box::new(move |bs| {
        let t = (*this)(bs)?;
        let u = f(t);
        Ok(u)
    })
}
```

Other operators, like `apply`, `expose` and `unit` follow similar approach.

In current form it is not too convenient to keep bunch of matchers around, so to avoid it I can just "glue" them together for composability into `Parser<T>`:

```rust
pub struct Parser<T> {
    f: Box<Matcher<T>>,
}

impl<T: 'static> Parser<T> {
    pub fn unit(f: Box<Matcher<T>>) -> Parser<T> {
        Parser { f }
    }

    pub fn init<F: Fn() -> T + 'static>(f: F) -> Parser<T> {
        Parser::unit(unit(f))
    }

    pub fn then<U: 'static>(self, that: Box<Matcher<U>>) -> Parser<(T, U)> {
        Parser::unit(chain(self.f, that))
    }

    pub fn map<U: 'static, F: Fn(T) -> U + 'static>(self, f: F) -> Parser<U> {
        Parser::unit(map(self.f, f))
    }
    ...
}
```

Same operators, but wrapped in `Parser::unit` actually allow convenient chaining.

At some point I've discovered that in order to define next matcher in a chain, glimpse of a current state of the accumulator is required (example: match the HTTP request body as exactly number of bytes in `Content-Length` header), I would need a way to look "into" the monad. Actually what I was looking for is a `flat_map` operator. I decided to name it `then_with` - just because I can.

```rust
pub fn expose<T: 'static, U: 'static, F: Fn(&T) -> Box<Matcher<U>> + 'static>(
    this: Box<Matcher<T>>,
    f: F,
) -> Box<Matcher<(T, U)>> {
    Box::new(move |bs| {
        let t = (*this)(bs)?;
        let g = f(&t);
        let u = (*g)(bs)?;
        Ok((t, u))
    })
}
```

```rust
impl<T: 'static> Parser<T> {
...
fn then_with<U: 'static, F: Fn(&T) -> Box<Matcher<U>> + 'static>(self, f: F) -> Parser<(T, U)> {
    Parser::unit(expose(self.f, f))
}
...
```

For example of usage, here is the implementation of HTTP request parser:

```rust
#[derive(Debug)]
struct Header {
    name: String,
    value: String,
}

#[derive(Debug, Default)]
struct Request {
    method: String,
    path: String,
    protocol: String,
    headers: Vec<Header>,
    content: Vec<u8>,
}

fn header_parser() -> Parser<Header> {
    Parser::init(|| vec![])
        .then(before(':'))
        .map(|(mut vec, val)| {
            vec.push(as_string(val));
            vec
        })
        .then(single(':'))
        .map(|(vec, _)| vec)
        .then(single(' '))
        .map(|(vec, _)| vec)
        .then(before('\n'))
        .map(|(mut vec, val)| {
            vec.push(as_string(val));
            vec
        })
        .then(single('\n'))
        .map(|(vec, _)| vec)
        .map(|vec| Header {
            name: vec[0].to_owned(),
            value: vec[1].to_owned(),
        })
}

fn request_parser() -> Parser<Request> {
    Parser::init(|| Request::default())
        .then(before(' '))
        .save(|req, bytes| req.method = as_string(bytes))
        .then(single(' '))
        .skip()
        .then(before(' '))
        .save(|req, bytes| req.path = as_string(bytes))
        .then(single(' '))
        .skip()
        .then(before('\n'))
        .save(|req, bytes| req.protocol = as_string(bytes))
        .then(single('\n'))
        .skip()
        .then(repeat(header_parser().into()))
        .save(|req, vec| req.headers = vec)
        .then(single('\n'))
        .skip()
        .then_with(|req| {
            let len: usize = get_header_value(req, "Content-Length")
                .map(|n| n.parse::<usize>().unwrap())
                .unwrap_or_default();
            bytes(len)
        })
        .save(|req, content| req.content = content)
}

// Checking if it actually works

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn http_request() {
        let text = r#"GET /docs/index.html HTTP/1.1
Host: www.nowhere123.com
Accept: image/gif, image/jpeg, */*
Accept-Language: en-us
Accept-Encoding: gzip, deflate
Content-Length: 8
User-Agent: Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1)

0123456
"#;
        let mut bs = text.to_string().into_stream();
        let req = request_parser().apply(&mut bs).unwrap();

        assert_eq!(req.method, "GET");
        assert_eq!(req.path, "/docs/index.html");
        assert_eq!(req.protocol, "HTTP/1.1");
        assert_eq!(req.content, b"0123456\n");
        assert_eq!(req.headers[0].name, "Host");
        assert_eq!(req.headers[0].value, "www.nowhere123.com");
        assert_eq!(req.headers[1].name, "Accept");
        assert_eq!(req.headers[1].value, "image/gif, image/jpeg, */*");
        assert_eq!(req.headers[2].name, "Accept-Language");
        assert_eq!(req.headers[2].value, "en-us");
        assert_eq!(req.headers[3].name, "Accept-Encoding");
        assert_eq!(req.headers[3].value, "gzip, deflate");
        assert_eq!(req.headers[4].name, "Content-Length");
        assert_eq!(req.headers[4].value, "8");
        assert_eq!(req.headers[5].name, "User-Agent");
        assert_eq!(req.headers[5].value, "Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1)");
    }
...
}
```

This is easy way to build clean, simple and composable parser combinators, glue them into monad, and define naive HTTP request parser, in pure Rust, without any dependencies. And this naive request parser event seems to work! At least test passes.

Current implementation is "stateless" in a way that if entity is not fully matched (e.g. not all chunks of a request were received), it returns error and next attempts will need to start over. For this reason, the client code should care about keeping the current read position in a stream and resetting the stream to that position if matching did not succeed.

The "stateful" parser could keep the aggregate and repeat matching from last position in a stream. I believe it would require to plug in the State monad somewhere, and allow returning something like `(State, Either<Error, Parser<T>>)` from parser.

The final code with implemented parsers for HTTP request and WebSocket frame are available on [github][code].

[mio]: https://github.com/sergey-melnychuk/mio-tcp-server
[code]: https://github.com/sergey-melnychuk/rust-parser-combinators
