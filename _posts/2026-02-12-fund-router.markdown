---
layout: post
title: "Deterministic Deposit Proxy in Solidity"
date: 2026-02-12 16:07:00 +0100
categories: solidity rust ethereum sepolia proxy
---

I built a small full-stack system for deterministic deposit addresses and automated treasury routing on Sepolia.

The core idea is simple:
- each user gets a deterministic proxy address (CREATE2),
- the app monitors those addresses for incoming ETH,
- once funded, the system routes value to a treasury through a permissioned router.

This post walks through the architecture, contract design, Rust backend, React UI, testing strategy, and deployment approach. Source code is on [GitHub](https://github.com/sergey-melnychuk/Of-Course-I-Knew-The-Address).

---

## Problem

For deposit-based flows, deterministic addresses are useful because you can:
- generate a destination address before deployment,
- share it immediately with users,
- deploy only when needed (or batch deploy later),
- keep address generation consistent across backend and on-chain logic.

The challenge is making this practical end-to-end:
- contract side (CREATE2 + minimal proxies + permissions),
- backend side (address generation, persistence, monitoring, routing),
- frontend side (visibility and actions),
- deployment side (easy demo + reproducible environment).

---

## Architecture

```mermaid
flowchart LR
    U[User] --> FE[React UI]
    FE --> API[Rust API: Axum]
    API --> DB[(SQLite)]
    API --> RPC[Sepolia RPC]
    API --> DEP[DeterministicProxyDeployer]
    DEP --> PX[CREATE2 Proxy]
    PX --> FR[FundRouter]
    FR --> T[Treasury]
    FR --> FS[FundRouterStorage]
```

### Main Flow

```mermaid
sequenceDiagram
    participant UI as React UI
    participant API as Rust Backend
    participant DB as SQLite
    participant RPC as Sepolia RPC
    participant DEP as Proxy Deployer
    participant PX as Proxy
    participant FR as FundRouter
    participant T as Treasury

    UI->>API: POST /api/deposits {user}
    API->>RPC: calculateDestinationAddresses(salt)
    API->>DB: INSERT deposit(user, salt, address, pending)
    API-->>UI: deposit metadata

    Note over PX: User sends ETH to predicted proxy address

    API->>RPC: poll balance(proxy)
    API->>DB: UPDATE deposits.balance

    UI->>API: POST /api/route
    API->>DEP: deployMultiple(salts for pending)
    API->>PX: transferFunds(...)
    PX->>FR: delegatecall
    FR->>T: ETH/ERC20 transfer
    API->>DB: UPDATE status=routed
    API-->>UI: routed tx hashes
```

---

## Smart Contract Design

### 1) DeterministicProxyDeployer

I use OpenZeppelin `Clones` to deploy EIP-1167 minimal proxies and `cloneDeterministic` for CREATE2.

Key points:
- `_deriveSalt(userSalt, msg.sender)` prevents cross-caller salt collisions.
- `calculateDestinationAddresses` uses the same derivation logic as deployment.
- `predictDeterministicAddress` makes off-chain prediction and on-chain deployment align.

```solidity
function _deriveSalt(
    bytes32 userSalt,
    address caller
) internal pure returns (bytes32) {
    return keccak256(abi.encodePacked(userSalt, caller));
}

function calculateDestinationAddresses(
    bytes32[] calldata salts
) external view returns (address[] memory out) {
    out = new address[](salts.length);
    for (uint256 i = 0; i < salts.length; i++) {
        bytes32 salt = _deriveSalt(salts[i], msg.sender);
        out[i] = FUND_ROUTER_ADDRESS.predictDeterministicAddress(salt);
    }
}
```

### 2) FundRouter + FundRouterStorage

`FundRouter` routes ETH and optional ERC20s, gated by storage permissions:
- `isAllowedCaller(msg.sender)` must be true,
- `isAllowedTreasury(treasury)` must be true,
- token and amount arrays must match.

`FundRouterStorage` is intentionally tiny and owner-controlled:
- permission bits: `0x01` caller, `0x02` treasury.

Why this split:
- router logic remains focused on transfer behavior,
- permission policy is isolated and replaceable.

```solidity
// FundRouterStorage — bitmask permission checks
function isAllowedCaller(address who) public view returns (bool) {
    return (permissions[who] & 0x01) == 0x01;
}

function isAllowedTreasury(address who) public view returns (bool) {
    return (permissions[who] & 0x02) == 0x02;
}
```

```solidity
// FundRouter — routing with permission gates
function transferFunds(
    uint256 etherAmount,
    address[] calldata tokens,
    uint256[] calldata amounts,
    address payable treasuryAddress
) external override {
    if (treasuryAddress == address(0)) revert ZeroTreasury();
    if (!STORAGE.isAllowedCaller(msg.sender)) revert NotAuthorizedCaller();
    if (!STORAGE.isAllowedTreasury(treasuryAddress))
        revert TreasuryNotAllowed();
    if (tokens.length != amounts.length) revert LengthMismatch();

    if (etherAmount > 0) {
        (bool ok, ) = treasuryAddress.call{value: etherAmount}("");
        if (!ok) revert EthSendFailed();
    }

    for (uint256 i = 0; i < tokens.length; i++) {
        uint256 amt = amounts[i];
        if (amt == 0) continue;
        bool ok = IERC20(tokens[i]).transfer(treasuryAddress, amt);
        if (!ok) revert Erc20TransferFailed();
    }
}
```

---

## Rust Backend Design

### API Surface

- `POST /api/deposits`
  - validates user address hex
  - computes salt (`keccak256(user)`)
  - predicts proxy address from deployed contract
  - stores row as `pending`
- `GET /api/deposits`
  - filter by user/salt/address/status
  - paging via `limit/offset`
- `POST /api/route`
  - selects `pending/proxied`
  - deploys missing proxies
  - routes balances to treasury
  - updates DB status and returns tx hashes

### Persistence Model

SQLite table stores:
- `user` (20-byte blob),
- `salt` (32-byte unique blob),
- `address` (20-byte unique blob),
- `status` (`pending`/`proxied`/`routed`),
- `balance` (32-byte blob, nullable),
- timestamps.

### Monitoring Loop

A background task polls balances every `POLL_BALANCE_DELAY` seconds and updates `balance`.

```rust
async fn poll_balances(state: Arc<AppState>) -> anyhow::Result<()> {
    let filters = db::DepositFilters {
        status: vec!["pending".to_string(), "proxied".to_string()],
        ..Default::default()
    };

    let deposits = db::query_deposits(&state.db, &filters).await?;
    let mut tx = state.db.begin().await?;

    for deposit in deposits {
        if let Ok(balance) = eth::get_balance(
            &state.config.sepolia_rpc_url,
            Address::from_slice(&deposit.address),
        )
        .await
        {
            sqlx::query("UPDATE deposits SET balance = ? WHERE id = ?")
                .bind(&balance[..])
                .bind(deposit.id)
                .execute(&mut *tx)
                .await?;
        }
    }
    tx.commit().await?;
    Ok(())
}
```

---

## Frontend Design

The frontend is intentionally operational and minimal:
- a single page dashboard,
- inline filtering (status/user/address),
- paging and refresh controls,
- actions per row (`Deploy`/`Route`),
- error modal and copy-to-clipboard UX.

Notable implementation details:
- same-origin API (`/api`) to keep deployment simple,
- copy fallback for non-HTTPS environments (`execCommand`) for demo hosting,
- typed data model for deposit rows.

---

## Testing Strategy

I added tests across all three layers.

| Layer | Tooling | Coverage |
|---|---|---|
| Contracts | Hardhat + Chai | 18 tests: permission checks, deterministic prediction, multi-deploy, duplicate salts, end-to-end deploy+fund+route |
| Backend utilities | Rust unit tests | hex decode/encode/validation, keccak vectors, determinism |
| Frontend | Vitest + Testing Library | `weiToEth` formatting, initial fetch rendering, create-deposit payload, error modal path |

Project-level command:

```bash
make test
```

---

## Deployment Approach

I optimized for one-command demo deployment:

- multi-stage Docker build:
  1. compile contracts,
  2. build frontend,
  3. compile Rust backend,
  4. run as a compact runtime image.
- frontend static output is embedded and served by Rust.
- deployment works on a clean Ubuntu/DigitalOcean instance with `docker compose up -d --build`.

I also used a fast path for low-powered VMs:
- build image locally for `linux/amd64`,
- `docker save | gzip`,
- upload tarball,
- `docker load` on the VM.

---

## Real Chain Validation (Sepolia)

I validated the system with actual Sepolia deployments and routing.

Deployed contracts:
- [FundRouterStorage](https://sepolia.etherscan.io/address/0x67979DE8C2F18FcC405415432000f7231AA8F12C)
- [FundRouter](https://sepolia.etherscan.io/address/0xD0d0F17Db168A74d6cb924F40cF062Fa40C857da)
- [DeterministicProxyDeployer](https://sepolia.etherscan.io/address/0x576a15Ff748b6F9BE74E7666E1A7c717AF096e5E)
- [Sample proxy](https://sepolia.etherscan.io/address/0xe24d719914b9e6bcfc95afe7ad8fd11ccbe6a101)
- [Sample route tx](https://sepolia.etherscan.io/tx/0xf4ca415a47f5500d6f6e1ebd7bb9cd4ae2d04a1e499d92173085fbc2857685da)

The important verification:
- predicted proxy address matched deployed proxy address,
- funded proxy value was routed to treasury,
- dashboard status transitioned `pending -> proxied -> routed`.

---

## Takeaways

The key design decision was making the on-chain contract the single source of truth for address prediction. The Rust backend calls `calculateDestinationAddresses` on the deployed contract rather than reimplementing CREATE2 derivation off-chain, so there's no risk of the two diverging. You could hand-craft proxy bytecode and replicate the address derivation in Rust — but that's working hard, not smart. One implementation to get right, not two to keep in sync.

The separation of `FundRouterStorage` from `FundRouter` felt like overkill at first for a project this size, but it paid off quickly when iterating on permission logic without redeploying the router.

If I were to revisit this, I'd spend more time on the routing step — right now it's a synchronous fan-out that would struggle with more than a handful of deposits. A proper job queue with retry semantics would be the natural next step.
