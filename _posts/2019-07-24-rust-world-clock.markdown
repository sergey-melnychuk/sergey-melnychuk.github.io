---
layout: post
title:  "World Clock in Rust"
date:   2019-07-24 22:24:42 +0200
categories: rust time time-zone chrono
---

Just simple console app that asks for a time zone and prints current time in provided timezone (or in local if no time zone was provided).

It requires two dependencies in `Cargo.toml`: [chrono][chrono] and [chrono-tz][chrono-tz]:

```
chrono = "0.4"
chrono-tz = "0.5"
```

The app just runs in a loop (Ctrl-C to break), asks for a time zone, and prints time. That's it!

```rust
extern crate chrono;
extern crate chrono_tz;

use std::io;
use std::io::BufRead;
use chrono::{Local, Utc};
use chrono_tz::Tz;

fn main() {
    let stdin = io::stdin();
    loop {
        let line = stdin.lock().lines().next().unwrap().unwrap();
        if line.is_empty() {
            let now = Local::now();
            println!("{}", now);
        } else {
            match line.parse::<Tz>() {
                Ok(tz) => {
                    let now = Utc::now();
                    println!("{}", now.with_timezone(&tz));
                },
                Err(_) => println!("(no such timezone)")
            }
        }
    }
}
```

For more details on date & time: [cookbook][cookbook].

[cookbook]:  https://rust-lang-nursery.github.io/rust-cookbook/datetime/parse.html
[chrono-tz]: https://github.com/chronotope/chrono-tz
[chrono]:    https://github.com/chronotope/chrono

