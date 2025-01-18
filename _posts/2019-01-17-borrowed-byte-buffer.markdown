---
layout: post
title:  "Borrowed Byte Buffer in Rust"
date:   2019-01-17 21:13:00 +0200
categories: rust, byte buffer
---
It started with re-implementation of gossip-based distributed membership protocol, this time in Rust. I was looking for a fast and simple way to put data into array of bytes, and to get data from an array of bytes. Something like Java's [ByteBuffer][java-bytebuffer].

Playing with memory allocation is fun, but it was not what I was looking for. That's why obvious decision was to read/write over an existing buffer, owned elsewhere. Apparently, this made excellent example of using Rust's lifetime checks, as that exsisting buffer isn't really owned by byte-buffer object but only referenced.

The full code for mutable (writable) and immutable (readable) borrowed byte buffer with unit-tests is under 300 LoC, awailable at [github][github-repo] and published at [crates][crate-page]. It's clearly not for production use, hence the version `0.0.1`. Code is not even close to elegant nor idiomatic, but it gets the job done.

```rust
pub struct ByteBuf<'a> {
    pos: usize,     // keep track of the current position in the array
    buf: &'a [u8],  // reference to the array with lifetime parameter 'a
}

pub struct ByteBufMut<'a> {
    pos: usize,         // same
    buf: &'a mut [u8],  // mutable reference with lifetime 'a
}
```

The `ByteBuf` allows reading u8, u16, u32 and u64, as well as `ByteBufMut` allows writing respectively 1, 2, 4 and 8 bytes at once. Getting words to bytes is pretty straightforward application of bit arithmetic.

```rust
impl<'a> ByteBuf<'a> {
    pub fn get_u8(&mut self) -> Option<u8> {
        // return one byte from the buffer, if it is awailable
    }
}

impl<'a> ByteBufMut<'a> {
    pub fn put_u8(&mut self, x: u8) -> usize {
        // put one byte in the buffer and return number of bytes written
    }
}
```

As long as there is no control/ownership over underlying buffer, the `get_*` methods return Option (reading after reaching end of buffer will return None) and `put_*` methods return actual number of bytes written to the buffer. It's not too convenient for sure, but it is the only way to go without managing and owning the underlynig buffer.

As a direction for improvement, the generic struct (de)serialization to/from binary woud be nice. Though reinventing the wheel doesn't make much sense when there are already awailable things like [simple binary encoding][sbe-github] and [serde serialization][serde-docs]

[java-bytebuffer]: https://docs.oracle.com/javase/8/docs/api/java/nio/ByteBuffer.html
[rust-lifetime]: https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html
[github-repo]: https://github.com/sergey-melnychuk/borrowed-byte-buffer
[crate-page]: https://crates.io/crates/borrowed-byte-buffer
[sbe-github]: https://github.com/real-logic/simple-binary-encoding
[serde-docs]: https://docs.serde.rs/serde/trait.Serializer.html
