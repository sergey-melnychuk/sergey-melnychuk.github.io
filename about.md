---
layout: page
title: "About"
permalink: /about/
---

Experienced software engineer passionate about functional programming, large-scale distributed systems and machine learning. Constantly pursuing self-improvement both personal and professional. With total 13+ (getting paid for programming since 2010) years in the industry, I am proficient mostly in Rust (5+ years of day-to-day experience), Java (10+ years) and Scala (7+ years) but also have meaningful experience (~1 year) also in Go, C++, C, Python and JavaScript.

Strong background in Computer Science and knowledge advanced data structures & algorithms (with Master's Degree in Computer Science since 2012), concurrent/parallel programming, distributed systems and algorithms, as well as TRIZ and Design Thinking approach allows designing and building clean, readable, testable, scalable software solutions for sophisticated problems. I don't stick to single language/tool/stack and keep generalist approach to pick a right tool for a job, taking into account requirements and trade-offs.

Professional experience with **Rust**:

- [Eiger](https://www.eiger.co) (1+ year):
  - development maintenance and of [Starknet](https://www.starknet.io/en) full node: [Pathfinder](https://github.com/eqlabs/pathfinder)
  - JSON-RPC code generation tool: [iamgroot](https://github.com/sergey-melnychuk/iamgroot)
  - Sandbox version of a new generation Starknet full node: [armada](https://github.com/sergey-melnychuk/armada)

- NEAR (via [GitCoin](https://www.gitcoin.co/))
  - AVL-tree implementation for [Near Rust SDK](https://docs.near.org/sdk/rust/introduction)
    - self-balancing binary tree to store key-value data
    - optimized for limited blockchain-specific runtime
    - provides API similart to [std::collections::BTreeMap](https://doc.rust-lang.org/std/collections/struct.BTreeMap.html)
  - [Pull Request](https://github.com/near/near-sdk-rs/pull/154)

- AdTech startup (Freelance, <1 year):
  - implementation of data enrichment pipeline for OpenRTB bid requests
  - implementation of RTB SSP components based on Cloudflare Workers

- IoT startup (Freelance, ~1 year): 
  - design and implementation of a low-level binary protocol for secure communications between IoT devices in Rust
  - implementation of a protocol client for ESP32 hardware in pure C

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

### <a name="star"></a> STAR cases

**S**ituation => **T**ask => **A**ction => **R**esult

---

#### Equinix, circa 2021

**Situation:** The project ("Job Launcher" - part of custom home-grown tech stack, whic is responsible for organizing and submitting Spark jobs to on-demand Amazon EMR clusters) passed over from previous team has signifant techincal and organizational problems.

**Task:** Take over project and keep up with SLA (Service Level Agreement) and SLC (Service Level Commitment). Resolve (a lot of) technical issues and mis-behaviours of the existing solution.

**Action:** The technical quality was too low (lots of anti-patterns, effectively lack of any meaningful tests), and estimation of fixing existing solution were higher that building similar one from scratch. So I made a decision for a complete rewrite with a focus on testability.

**Result:** The new software was delivered in 3 sprints (two weeks each). New software solved a lot of problems the old one had (multi-env support, testability, observability, opportunity to reuse it across the organization).

---

#### Google, circa 2019

**Situation:** I am assigned a complex feature for Autopilot project - vertical & horizontal auto-scaler of Borg's (Googl's internal job running serviec, basically a grandfather of Kubernetes) jobs resources (number of instances per region, CPU, memory, disk).

**Task:** Reduce "wasted" (over-commited) resources for complex multi-level redundancy setups. Example: a job needs N+2 redundancy per continent (AMER, EMEA, APAC) level and N+1 redundancy per metro (WAW, TLV, etc) level.

**Action:** Design a clean and complete data model, provide extensive test coverage for edge-cases (complex multi-level redundancy setups), using specific modeled test-cases verify that "waste" indeed exists and can be reduced. Design a 2-phase roll-out plan, taking into account that resource limits assigned by updated peers are not over-compensated of yet-to-be-updated ones. Implement the algorithm, get it reviewed and approved, deliver the update over two roll-out "waves".

**Result:** The global (in Google-scale) rollout of updated algorithm succeeded. The savings were validated using A/B tests of the Autopilot's assigned resource limits reported by telemetry. Advanced telemetry queries allowed to simulate limit assigning reporting, thus enable A/B testing with clear "before" and "after" states.

---

#### StrikeAd/Sizmek, circa 2017

**Situation:** The critical business feature (Native Ads support) delivery on a very tight deadline (single 2-week sprint) is complicated by enormous technical debt accumulated on front-end side.

**Task:** As a Team-Lead of a feature-team, I am responsible for delivering feature to production on time (it was promised already by the business development team).

**Action:** Organize and coordinate front-end "tribe" with plan to "pay off" the technical debt in iterations, while allowing some amout of it slip into the closest release. Deliver the remaining components of the feature on time without disruption (back-end, test coverage, analytics, reporting).

**Result:** Critical business feature delivered on time. Technical debt for the front-end is organized and managed into a clean and actionable plan, that does not disrupt the feature delivery process.

---
