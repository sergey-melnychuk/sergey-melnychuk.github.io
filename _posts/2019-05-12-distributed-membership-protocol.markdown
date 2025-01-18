---
layout: post
title:  "Distributed Membership Protocol"
date:   2019-05-12 19:30:42 +0200
categories: distributed-algorithms failure-detection gossip-protocol distrbuted-systems rust
---
Once I had a long winter evening to spare, and I was thinking, why don't I implement a distributed membership and failure-detection protocol in Rust?

TL;DR: [github][code]

At the moment, it is rather example than usable library, but I have great plans for making it into efficient and usable tool for decentralized coordination.

Being already pretty [familiar][distributed-algorithms-github] with distributed membership and failure detection protocols, goal was mostly to write some Rust code and to get familiar with `std::net` package. The UDP protocol support is rather straightforward:

```rust
let socket = UdpSocket::bind(SocketAddr::from(([0, 0, 0, 0], port)))
        .expect("failed to bind");
socket.set_read_timeout(Some(Duration::from_millis(read_timeout_millis)))
        .expect("failed to set read timeout");

let mut buf = [0u8; 1024];
loop {
        let res = socket.recv_from(&mut buf);
        if res.is_ok() { /* do something useful */ }
}
```

The idea of distributed membership and failure detection is simple - if a node doesn't hear from peer for some time, such peer can be considered as failed. Thus for a peer to not be considered as failed, it must spread it's good and healthy status to other peers. I decided to start with all-to-all heartbeat propagation and with constant timeouts for marking peers as unreachable or failed. [Gossip protocol][gossip-protocol] is a well defined concept after all.

The structure of a peer is very simple, and keeps address, most recent received heartbeat and timestamp when this most recent heartbeat was received. This timestamp is used later to reason about peer state: if's too old, then reasonable assumption is made that such peer has failed.

```rust
#[derive(Debug, Copy, Clone)]
pub struct Record {
    pub addr: Addr,
    pub beat: u64,
    pub time: u64,
}
```

The protocol itself is very simple as well, and consist of only two messages:
- *Join* is sent to one of peers when newly started node want's to join existing cluster
- *List* is sent to share state of node's peers with those peers

```rust
#[derive(Debug)]
pub enum Message {
    Join(Record),
    List(Vec<Record>)
}
```

Sending list of all peers to all peers does sound like an overhead, and it is. So for large deployments it makes sense to actually apply 'gossip' approach for sharing state, sending only fraction of peers' states (say, 3) to other peers. Such infectious gossip will converge in linear time (can't remember the proof link, just use DuckDuckGo), keeping traffic linear as well (each peer sends constant-size messages). This will look much better, and will reduce quadratic (each peer sends to each) throughput and qubic (!) bandwith (each message contains all peers).

Such example of distributed membership can be run locally, after building with `caro build`.

```bash
# Terminal 1
$ ./gossip-peer 12000
listening at: 12000
# Now start another node in Terminal 2
append: Record { addr: 127.0.0.1:12000, beat: 0, time: <timestamp> }
# Now stop this node by Ctrl-C
^C
$ # Now check Terminal 2 if it detected stopped node
```

```bash
# Terminal 2
$ ./gossip-peer 12001 127.0.0.1:12000
listening at: 12001
append: Record { addr: 127.0.0.1:12001, beat: 1, time: <timestamp> }
# Now check Terminal 1 - it must detect new node
remove: Record { addr: 127.0.0.1:12000, beat: 42, time: <timestamp> }
# Unresponsive node detected and removed from list
^C
$ 
```

As for further plans of [gossip-peer][code] development, now I can imagine doing such (probably) useful things:
- Extensive unit- and system-test coverage :)
- Switch from `std::net` to [tokio][tokio] or [actix][actix]
- Automatic switch from all-to-all to gossip when cluster reach certain size
- Support adaptive timeout for failure detection to address network congestion condition
- Support decentralized service discovery by adding `tag` property for each peer
- Support decentralized load balancing among peers with the same `tag`
- Support multicast-based peers lookup (won't work in the cloud though)
- Implement application-level broadcast/multicast (can be used for cloud deployments)
- Support simple distributed replicated in-memory key-value database (like [consul][consul])
- Provide low-footprint library to include peer-awareness in existing network applications
- Some other interesting distributed wheel to re-invent!

[distributed-algorithms-github]: https://github.com/sergey-melnychuk/distributed-algorithms
[gossip-protocol]: https://www.consul.io/docs/internals/gossip.html
[code]: https://github.com/sergey-melnychuk/gossip-peer
[tokio]: https://tokio.rs
[actix]: https://actix.rs
[consul]: https://www.consul.io

