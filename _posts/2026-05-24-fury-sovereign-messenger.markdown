---
layout: post
title: "FURY: sovereign messenger in Rust"
date: 2026-05-24 12:00:00 +0200
categories: nostr rust tor cryptography privacy
---

TL;DR: [FURY](https://github.com/sergey-melnychuk/fury) — pure-Rust encrypted messenger. One BIP-39 mnemonic is your identity. Every byte goes through embedded Tor. Every message is NIP-44 v2 encrypted. No account, no phone number, no server you don't own.

### Why

The usual privacy messengers make at least one uncomfortable trade-off. Signal requires a phone number and runs its own server infrastructure. Session has no phone number and a seedphrase identity, but it built its own relay network from scratch. Briar is Tor-native but has no open relay network to speak of. None of them give you a single seed that simultaneously controls your chat identity, your Ethereum wallet, and your Bitcoin wallet.

The Nostr ecosystem already has thousands of relays and hundreds of thousands of users. NIP-44 v2 is a well-specified, auditable encryption standard. Arti — the Tor Project's official pure-Rust Tor implementation — ships as a library, meaning Tor can be embedded inside a binary with no system daemon required. These three things together felt like a natural fit.

FURY is the result: a Rust-native, Tor-native, Nostr-native messenger where your 12-word mnemonic is the only secret you need to keep.

### Identity from a mnemonic

The starting point is a BIP-39 mnemonic — the same 12 or 24 words used for Ethereum and Bitcoin wallets. FURY derives three separate keys from it using standard BIP-32 HD derivation:

| Key | Path | Purpose |
|---|---|---|
| Nostr | `m/44'/1237'/0'/0/0` | chat identity (NIP-06) |
| Ethereum | `m/44'/60'/0'/0/0` | EVM wallet |
| Bitcoin | `m/44'/0'/0'/0/0` | Bitcoin / Lightning |

Using BIP-32 child derivation rather than slicing the raw seed means the keys are portable — any NIP-06-compliant Nostr client can derive the same identity from the same words. The Nostr key uses BIP-340 Schnorr signatures over secp256k1, which is what NIP-01 requires.

```rust
pub fn new(mnemonic: SecretString) -> FuryResult<Self> {
    let mut raw: [u8; 32] = {
        let phrase = mnemonic.expose_secret();
        let mnemonic_obj = bip39::Mnemonic::parse(phrase)?;
        let seed = mnemonic_obj.to_seed("");
        let path: DerivationPath = NOSTR_PATH.parse()?;
        let child = XPrv::derive_from_path(seed, &path)?;
        child.private_key().to_bytes().into()
    };

    let key_bytes = Box::new(raw);
    raw.zeroize(); // scrub stack copy

    let locked = unsafe { memsec::mlock(key_bytes.as_ref().as_ptr() as *mut _, 32) };
    if !locked {
        return Err(FuryError::HardwareLock);
    }

    Ok(Self { key_bytes })
}
```

A few things worth noting here. The 512-byte BIP-39 seed and all intermediate derivation buffers live only in the inner block scope; they are dropped (and zeroized) before the private scalar ever leaves the function. The scalar itself goes onto the heap in a `Box<[u8; 32]>` — a stable address, unlike the stack which can be copied between frames by the compiler. Then `mlock(2)` tells the kernel not to page that page to swap, and not to include it in core dumps.

`FuryIdentity::new` returns `Err(FuryError::HardwareLock)` if the kernel refuses `mlock` — it never silently downgrades to an unprotected key. On `Drop`, `munlock` is called before `zeroize`: the page must be unlocked before the allocator can reuse it, so the zero-write is not optimized away on a still-locked page.

The `SigningKey` is never stored in the struct. It is reconstructed from the locked bytes for each `sign()` call and dropped immediately, minimizing the window during which key material exists outside the locked page.

### NIP-44 v2 encryption

NIP-44 is the Nostr direct-message encryption standard. Version 2 replaced the earlier (broken) XSalsa20 scheme with a proper authenticated construction:

```
conversation_key = HKDF-extract(salt="nip44-v2", ikm=ECDH(priv_a, pub_b))
nonce            = random 32 bytes
[chacha_key(32) | chacha_nonce(12) | hmac_key(32)] = HKDF-expand(conv_key, nonce, 76)
padded           = len_prefix(2 BE) || plaintext || zeros → calc_padded_len(len)
ciphertext       = ChaCha20(chacha_key, chacha_nonce, padded)
mac              = HMAC-SHA256(hmac_key, nonce || ciphertext)
payload          = 0x02 || nonce || ciphertext || mac  → base64
```

The conversation key is computed once per (sender, recipient) pair using ECDH on secp256k1. The nonce is per-message, so a fresh ChaCha20 key is derived for every message. Decryption verifies the HMAC before touching the ciphertext (authenticate-then-decrypt).

The padding scheme is worth mentioning: short messages are padded to at least 32 bytes, longer messages to the next boundary in a chunk size that grows with the message length. This prevents an observer from inferring message length even knowing the ciphertext size.

```rust
fn calc_padded_len(len: usize) -> usize {
    if len <= 32 {
        return 32;
    }
    let next_power = 1usize << ((len - 1).ilog2() + 1);
    let chunk = if next_power <= 256 { 32 } else { next_power / 8 };
    chunk * ((len - 1) / chunk + 1)
}
```

The implementation is verified against the [official NIP-44 test vectors](https://github.com/paulmillr/nip44), which cover conversation key derivation, full encrypt/decrypt round-trips, and Unicode edge cases.

### Tor transport via Arti

Every relay connection goes through Arti — the Tor Project's pure-Rust Tor implementation — embedded directly in the FURY binary. No system daemon, no `tor` package, no configuration file. The binary carries the entire Tor consensus engine.

```rust
pub async fn bootstrap_tor() -> FuryResult<TorClient<PreferredRuntime>> {
    TorClient::create_bootstrapped(TorClientConfig::default())
        .await
        .map_err(|e| FuryError::Network(e.to_string()))
}
```

The first bootstrap takes about 10 seconds while Arti downloads the Tor consensus and builds circuits. Subsequent runs reuse the on-disk cache.

`RelayClient` wraps the Tor-routed WebSocket in a channel-based design: a background task owns the socket and bridges it to an `mpsc` sender (outbound) and a `broadcast` channel (inbound). This lets `publish` and `subscribe` share a single connection concurrently without any external locking:

```rust
tokio::select! {
    Some(frame) = out_rx.recv() => {
        if sink.send(Message::Text(frame)).await.is_err() {
            break;
        }
    }
    msg = stream.next() => {
        match msg {
            Some(Ok(Message::Text(t))) => { let _ = in_tx2.send(t.to_string()); }
            Some(Ok(Message::Close(_))) | None => break,
            // ...
        }
    }
}
```

The privacy invariant is enforced structurally: `RelayClient::connect` requires a `TorClient` argument. There is no `connect_direct()`. The relay never sees the user's IP address.

### The chat demo

`fury-chat` is the end-to-end demo: two terminals, a live Nostr relay, NIP-44 encrypted messages through Tor.

```bash
# Terminal A
FURY_MNEMONIC="abandon abandon abandon ..." \
FURY_RECIPIENT="17162c921dc4..." \
FURY_RELAY="wss://nos.lol" \
cargo run -p fury-chat

# Terminal B
FURY_MNEMONIC="leader monkey parrot ring ..." \
FURY_RECIPIENT="e8bcf38236..." \
FURY_RELAY="wss://nos.lol" \
cargo run -p fury-chat
```

These mnemonics are the NIP-06 standard test vectors — safe for demo use. The identity is derived, Tor bootstraps, the relay connection goes up, and the two sides can exchange encrypted messages. The relay sees signed NIP-01 events with encrypted content. It does not see plaintext. It does not see IPs.

The stdin loop is straightforward: read a line, encrypt with the conversation key, wrap in a NIP-01 event signed with BIP-340 Schnorr, publish. The receive side subscribes to kind:4 events from the peer's pubkey tagged with our pubkey, decrypts, prints.

### What the relay sees

A useful way to think about the privacy model is to enumerate exactly what each party learns:

- **The relay** sees your signed Nostr pubkey, the timestamp, and the encrypted content blob. It cannot read the message. It cannot learn your IP (Tor hides it). It knows you communicate with whoever your `#p` tag points at — but Nostr pubkeys are designed to be public identifiers.
- **Tor** sees circuit construction metadata. Individual Tor nodes see only the next hop. Entry nodes know your IP but not your destination. Exit nodes know your destination but not your IP.
- **Apple/Google** (push notifications, when mobile lands) will receive only `{"relay":"nos.lol"}` — no message content, no sender identity, no event count. The device wakes, connects through its own Arti stream, and fetches events itself.

### What's done and what's next

The core stack is working: identity derivation with NIP-06, NIP-44 v2 encryption with official test vectors, Tor transport via Arti, and the two-terminal chat demo over a live relay.

What's missing: encrypted mnemonic storage at rest (Argon2id + ChaCha20-Poly1305, macOS Keychain for the wrapping key), MLS group messaging (RFC 9420 via `openmls` — the same protocol used by Signal for groups), push notifications, and eventually a Tauri desktop app and UniFFI mobile bindings.

The path to mobile is deliberate: `fury-core` is a headless library with zero platform dependencies. UniFFI generates Kotlin and Swift bindings from the same Rust code that runs the CLI. Platform code handles only UI and OS integration; all cryptography stays in one place.

The project is at a point where the hard parts of the privacy stack are done. The relay never sees your IP. The relay never sees your plaintext. Your identity requires no account registration. The same 12 words that open your chat also control your Ethereum and Bitcoin keys.

Full code: [github.com/sergey-melnychuk/fury](https://github.com/sergey-melnychuk/fury)
