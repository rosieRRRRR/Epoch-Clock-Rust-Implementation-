# Epoch Clock (Rust Implementation)

> A Rust-native trustless time logic tool for Bitcoin contracts. Designed by [Rosie](https://github.com/rosieRRRRR) as the original creator of the Epoch Clock, with extensions for L1 coordination, covenant expiry, and BitVM integration.

```text
epoch = floor((reference_timestamp - genesis_timestamp) / epoch_interval_seconds)
```

---

## TL;DR

Epoch Clock is a simple, verifiable way to synchronise Bitcoin contracts over time using only confirmed timestamps. No oracles. No assumptions. Just consensus.

---

## Specification

The Epoch Clock protocol defines a deterministic time epoch based solely on Bitcoin-anchored timestamps.

### Required Fields

- `protocol`: must be "epochclock"
- `version`: must follow semver format (e.g. "0.1")
- `genesis_timestamp`: integer UNIX time (seconds), must be from a confirmed Bitcoin block or transaction
- `epoch_interval_seconds`: integer ≥ 1

### Epoch Calculation

```text
epoch = floor((reference_timestamp - genesis_timestamp) / epoch_interval_seconds)
```

---

## Trust Model

Epoch Clock (Rust) depends only on:

- A confirmed `genesis_timestamp`
- A fixed interval (`epoch_interval_seconds` ≥ 1)
- A Bitcoin-confirmed `reference_timestamp`

All logic is:

- Deterministic
- Fully local
- Verifiable using only Bitcoin consensus data

No internet clocks, no external validators. All timestamps must originate from consensus-confirmed Bitcoin blocks or transactions.

### Assumptions

- Timestamps are extracted from consensus sources like block headers or transaction locktimes
- Malicious miners could potentially manipulate timestamps by a small margin (e.g. 2-hour drift rule), but this is bounded by Bitcoin consensus rules
- Contracts must verify that `reference_timestamp` is valid within Bitcoin context
- Developers are advised to choose sufficiently long `epoch_interval_seconds` to mitigate borderline timestamp manipulation at epoch boundaries

### L1 Compatibility

Epoch Clock operates entirely on Bitcoin Layer 1. No consensus changes are required.

---

## Key Features (Rust Version)

- ✅ Native Rust library and CLI
- ✅ Stateless, trustless, and deterministic
- ✅ No external clocks or APIs required
- ✅ Bitcoin-only timestamp validation
- ✅ Compatible with L1, PSBTs, and BitVM
- ✅ Formal config schema (JSON)
- ✅ Built-in error handling and test cases
- ✅ Optional config inscription support
- ✅ Open for integration into covenant tools and smart contract frameworks

---

## What is an Epoch?

An **epoch** is a deterministic time window defined by:

- A confirmed `genesis_timestamp` (UNIX seconds)
- An `epoch_interval_seconds` (e.g. 3155760000 for 100 Julian years)

```text
epoch = floor((reference_timestamp - genesis_timestamp) / epoch_interval_seconds)
```

Epoch 0 begins at the genesis timestamp. Epoch 1 begins after one interval. All further epochs follow at fixed steps. This model enables contract expiry and activation logic without external dependencies.

---

## Build

```bash
git clone https://github.com/rosieRRRRR/EpochClock.git
cd EpochClock
cargo build --release
```

---

## CLI Usage

```bash
./target/release/epochclock config.json 1860000000
```

- `config.json`: JSON file defining the Epoch Clock parameters
- `1860000000`: Reference timestamp (must originate from Bitcoin)

### Example Output

```
EpochClock ✅  Calculated Epoch: 1 (based on input timestamp: 1860000000)
```

### Help

```bash
./epochclock --help
Usage: epochclock <config_path> <timestamp>
```

---

## Config Schema

```json
{
  "protocol": "epochclock",
  "version": "0.1",
  "genesis_timestamp": 1710000000,
  "epoch_interval_seconds": 3155760000
}
```

This file can be embedded directly in a contract or inscribed publicly. Timestamps must be valid UNIX seconds from Bitcoin-confirmed data. Epoch interval must be ≥ 1.

---

## Verifying Genesis Timestamp

Use a block explorer or full node to confirm the genesis timestamp:

```bash
bitcoin-cli getblockheader <blockhash>
```

This ensures the genesis reference is auditable and anchored to consensus. Any timestamp used must be directly extractable from Bitcoin.

---

## Timestamp Sources

Valid Bitcoin-consensus-confirmed timestamps include:

- `nTime` field in a mined block header
- `locktime` from a confirmed transaction
- `OP_CLTV`-enforced script values
- Constants embedded in BitVM circuits

Timestamps **must not** come from:

- Mempool-only transactions
- Local clocks or devices
- External APIs or NTP servers

---

## 🥉 Real-World Use Cases

Epoch Clock is suitable for any Bitcoin logic that requires coordination over time **without relying on system clocks or oracles**. Example applications include:

### • Trustless Loan Expiry

Lock Bitcoin with a script or BitVM circuit that becomes spendable only in a specific epoch. If the borrower doesn’t repay before the epoch changes, the lender can reclaim funds.

### • Cross-Contract Coordination

Multiple contracts can reference the same `genesis_timestamp` and `epoch_interval_seconds` to synchronise behaviour - like vaults, escrows, or rollouts - all triggered by Bitcoin-confirmed timestamps.

### • Token Vesting or Unlock Schedules

Define when tokens can be spent, distributed, or migrated based purely on epoch transitions. No servers required.

### • Rate Limiting or Fair Drop Windows

Run on-chain time-based controls for mints, auctions, or claims that can’t be botted or gamed through fast mempool relays.

### • BitVM Execution Branching

Use the current epoch as a conditional input to determine which tree or logic path executes inside a BitVM circuit.

### • Expiry Enforcement

- Compare the current epoch to a preset epoch
- Enforce spend conditions via script, PSBT, or BitVM tree structure

---

## Usage Examples

- Enforce spending only in a target epoch via script checks
- Combine with BitVM to lock execution paths to specific epochs
- Compare `current_epoch` against embedded contract expiry logic

---

## BitVM Integration

To integrate Epoch Clock logic into a BitVM circuit:

- Use the `epoch` value as a **gate condition** in your circuit logic
- Store `genesis_timestamp` and `epoch_interval_seconds` as constants within the circuit or agree to them off-chain via signed config
- Compute `current_epoch` off-chain and feed it into the circuit input
- Prove correctness of the `epoch` value by demonstrating:

```text
floor((ts - genesis) / interval) == expected_epoch
```

BitVM execution paths can then commit to epoch-specific branches:

- Tree A: Loan repayment path (epoch 0)
- Tree B: Liquidation path (epoch ≥ 1)

If the prover can only generate a valid trace for the matching epoch, enforcement remains trustless.

You may choose to embed the config hash or the full logic inline with the BitVM circuit. This adds expiration and synchronisation logic without soft forks or external time feeds.

To learn more about BitVM, visit the [BitVM repository by Robin Linus](https://github.com/RobinLinus/bitvm).

---

## Test Cases

Also available in `tests/epoch.rs`

```bash
# Case 1
config.genesis_timestamp = 1710000000
config.interval = 3155760000
reference = 1860000000  =>  epoch = 0

# Case 2
config.genesis_timestamp = 1000000000
config.interval = 100000000
reference = 1100000000  =>  epoch = 1
```

---

## Edge Cases

- If `reference_timestamp < genesis_timestamp`, returns `EpochError::TimestampBeforeGenesis`
- Ensure subtraction and division do not overflow
- All values must be unsigned positive integers
- Choose sufficiently long epochs to avoid epoch drift exploitation by miners

---

## Library Use

```rust
let epoch = epochclock::calculate_epoch(reference_ts, &config)?;
```

Returns `Result<i64, EpochError>`. Fully deterministic.

---

## Integrity Check

```bash
sha256sum config.json
# SHA256(config.json): e1f3a9a6f4a0b237781ea8e91cb70fffb72f6341c7e3e88d1a25a31814e9bde6
```

This checksum may be embedded in:

- PSBT metadata
- BitVM gate trees
- Covenant scripts

---

## Optional Public Config

You **may** publish or inscribe a `config.json`, but this is not required. All logic is local and trustless.

Reasons to inscribe:

- Public coordination across multiple contracts
- Shared reference for epoch synchronisation
- Decentralised audit trail

Example:

```
Inscription ID: <insert if used>
SHA256(config.json): e1f3a9a6f4a0b237781ea8e91cb70fffb72f6341c7e3e88d1a25a31814e9bde6
```

---

## License

MIT

---

## Maintainer

[Rosie](https://github.com/rosieRRRRR)
