---
title: SDK Overview
sidebar_position: 1
---

# Kernel SDK Overview

The `kernel-sdk` crate provides the canonical interface for developing agents that execute inside the zkVM guest environment.

## Design Principles

1. **Stability** - Versioned interface, backwards-compatible within major versions
2. **Minimalism** - Agents receive only what they strictly need
3. **Isolation** - Agents cannot access forbidden APIs or kernel internals
4. **Determinism** - Agent execution must be fully deterministic
5. **Auditability** - Agent behavior must be inspectable and reproducible

## Crate Attributes

```rust
#![no_std]
#![forbid(unsafe_code)]
#![deny(clippy::std_instead_of_alloc)]
#![deny(clippy::std_instead_of_core)]
```

The SDK is `no_std` and forbids unsafe code. Agents inherit these constraints.

## Module Structure

```
kernel-sdk/
├── src/
│   ├── lib.rs          # Crate root with re-exports
│   ├── agent.rs        # AgentContext and AgentEntrypoint
│   ├── types.rs        # ActionV1, AgentOutput
│   ├── math.rs         # Checked/saturating arithmetic
│   ├── bytes.rs        # Binary reading/writing helpers
│   └── prelude.rs      # Convenient imports
```

## Using the SDK

Add to your `Cargo.toml`:

```toml
[dependencies]
kernel-sdk = { path = "../sdk/kernel-sdk" }
kernel-core = { path = "../protocol/kernel-core" }
```

Import the prelude for common types:

```rust
use kernel_sdk::prelude::*;
```

## The Prelude

The prelude exports commonly used items:

| Category | Items |
|----------|-------|
| Context | `AgentContext` |
| Types | `ActionV1`, `AgentOutput`, `MAX_ACTIONS_PER_OUTPUT`, `MAX_ACTION_PAYLOAD_BYTES` |
| Action Constants | `ACTION_TYPE_ECHO`, `ACTION_TYPE_OPEN_POSITION`, `ACTION_TYPE_CLOSE_POSITION`, `ACTION_TYPE_ADJUST_POSITION`, `ACTION_TYPE_SWAP` |
| Constructors | `echo_action`, `open_position_action`, `close_position_action`, `adjust_position_action`, `swap_action` |
| Decode Helpers | `decode_*_payload`, `DecodedOpenPosition`, `DecodedAdjustPosition`, `DecodedSwap` |
| Math | `checked_add_u64`, `checked_mul_div_u64`, `apply_bps`, `calculate_bps`, `BPS_DENOMINATOR` |
| Bytes | `read_u32_le`, `read_u64_le`, `read_bytes32`, `read_u32_le_at`, etc. |
| Alloc | `Vec` (NOT `vec![]` macro) |

## Core Types

### AgentContext

Contains execution context passed to the agent:

```rust
pub struct AgentContext<'a> {
    pub protocol_version: u32,
    pub kernel_version: u32,
    pub agent_id: &'a [u8; 32],
    pub agent_code_hash: &'a [u8; 32],
    pub constraint_set_hash: &'a [u8; 32],
    pub input_root: &'a [u8; 32],
    pub execution_nonce: u64,
    pub opaque_inputs: &'a [u8],
}
```

### AgentOutput

The agent's return value containing actions:

```rust
pub struct AgentOutput {
    pub actions: Vec<ActionV1>,
}
```

### ActionV1

A single action to be executed:

```rust
pub struct ActionV1 {
    pub action_type: u32,
    pub target: [u8; 32],
    pub payload: Vec<u8>,
}
```

## Action Types

| Constant | Value | Description |
|----------|-------|-------------|
| `ACTION_TYPE_ECHO` | `0x00000001` | Test/echo action |
| `ACTION_TYPE_OPEN_POSITION` | `0x00000002` | Open trading position |
| `ACTION_TYPE_CLOSE_POSITION` | `0x00000003` | Close position |
| `ACTION_TYPE_ADJUST_POSITION` | `0x00000004` | Modify position |
| `ACTION_TYPE_SWAP` | `0x00000005` | Asset swap |

## Helper Functions

### Action Constructors

```rust
// Create an echo action
let action = echo_action(target, payload);

// Create an open position action
let action = open_position_action(
    target,
    asset_id,
    notional,
    leverage_bps,
    direction,  // 0 = Long, 1 = Short
);

// Create a close position action
let action = close_position_action(target, position_id);

// Create an adjust position action
let action = adjust_position_action(
    target,
    position_id,
    new_notional,
    new_leverage_bps,
);

// Create a swap action
let action = swap_action(target, from_asset, to_asset, amount);
```

### Math Helpers

```rust
// Checked arithmetic (returns None on overflow)
let sum = checked_add_u64(a, b)?;
let product = checked_mul_u64(a, b)?;
let quotient = checked_div_u64(a, b)?;

// Compound operations
let result = checked_mul_div_u64(a, b, denom)?;

// Basis points
let fee = apply_bps(amount, 100)?;  // 1% fee
let pct = calculate_bps(numerator, denominator)?;
```

### Byte Helpers

```rust
// Fixed-offset readers
let value = read_u32_le(bytes, offset)?;
let hash = read_bytes32(bytes, offset)?;

// Cursor-style readers (advance offset)
let mut offset = 0;
let value = read_u64_le_at(bytes, &mut offset)?;
let hash = read_bytes32_at(bytes, &mut offset)?;
```

## Forbidden Behavior

Agents MUST NOT:

| Forbidden | Reason |
|-----------|--------|
| `std::time` | Non-deterministic |
| `rand` / randomness | Non-deterministic |
| `std::fs`, `std::net` | I/O forbidden in guest |
| Syscalls / host functions | Isolation violation |
| Kernel internals | Isolation violation |
| Unbounded memory | Resource exhaustion |
| Floating-point operations | Non-deterministic across platforms |
| HashMap, HashSet | Non-deterministic iteration order |

## Size Limits

| Constant | Value | Description |
|----------|-------|-------------|
| `MAX_ACTIONS_PER_OUTPUT` | 64 | Maximum actions per output |
| `MAX_ACTION_PAYLOAD_BYTES` | 16,384 | Maximum payload per action |
| `MAX_AGENT_INPUT_BYTES` | 64,000 | Maximum opaque input size |

## Next Steps

- [Writing an Agent](/sdk/writing-an-agent) - Build your first agent
- [Constraints](/sdk/constraints-and-commitments) - Understanding constraint enforcement
- [Testing](/sdk/testing) - Test your agent at multiple levels
