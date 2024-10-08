# ParallelChain Request for Comments 1 (PRFC 1)

| PRFC | Title | Author | Version | Date First Published |
| --- | ----- | ---- | --- | --- |
| 1   | Fungible Token Standard | ParallelChain Lab | 3 | July 23rd, 2022 | 

## Summary

ParallelChain RFC 1 defines a standard interface for fungible tokens implemented as ParallelChain smart contracts. “Fungible tokens” is taken here to have the same meaning as in Ethereum's ERC 20, namely a set of identical, transferable entities. XPLL, for example, is a fungible token.

A standard contract interface for fungible tokens allows more seamless interoperability since applications can make the simplifying assumption that all PRFC 1-implementing contracts always export the same, named set of Methods (they may export more).

The below sections list the set of methods that all smart contracts that want to be PRFC 1-compliant must implement, as well as the behavior that each method must exhibit. Required behavior involves emitting certain events. These are also listed and described.

## Notes

- The following uses syntax from Rust (version 1.59.0).
- The data type `PublicAddress` is the type alias to a 32-byte slice `[u8; 32]`. 
- The term `calling_account` refers to the account that invokes the method.

## Required types

A borsh-serializable structure to represent a Token.

```rust
struct Token {
    name: String,
    symbol: String,

    // the number of decimals that should be divided from a token amount (such as that returned by the method
    // 'balance_of') to get its user representation. i.e., `n` means to divide the token amount by `10^n`.
    decimals: u8,

    // ‘Supply’ here means the sum of token balances over *all* addresses at any given point in time.
    // Total supply may increase when new tokens are minted, or decrease when tokens are burned.
    total_supply: u64
}
```

## Required Views 

### token

```rust
fn token() -> Token
```

Returns information about the Token implemented by this contract.

### allowance

```rust
fn allowance(owner: PublicAddress, spender: PublicAddress) -> u64
```

Returns the amount of tokens that the `spender` can currently spend on behalf of the `owner`.

`allowance` must never panic. If `owner` does not have an allowance, this function must return 0.

### balance_of

```rust
fn balance_of(address: PublicAddress) -> u64
```

Queries the amount of tokens owned by the account identified by `address`.

`balance_of` must never panic. If `address` does not own any tokens, this function must return 0.


## Required Calls

### transfer
```rust
fn transfer(to_address: Option<PublicAddress>, amount: u64)
```

Transfers tokens to an account identified by `to_address` from the `calling_account`. If `to_address` is None, this burns the amount.

`transfer` must panic if `balance_of(calling_account)` < `amount`.

Log `Transfer` must be emitted if `transfer` is successful.


### transfer_from
```rust
fn transfer_from(from_address: PublicAddress, to_address: Option<PublicAddress>, amount: u64)
```

Transfers tokens to an account identified by `to_address` on behalf of the owner (`from_address`). If `to_address` is None, this burns the amount.

`transfer_from` must panic if `allowance(from_address, calling_account)` < `amount`.

Log `Transfer` must be emitted if `transfer_from` is successful. Note that the topic of the `Transfer` log must contain the owner's address (`from_address`), not the address of the `calling_account`.

### set_allowance
```rust
fn set_allowance(spender: PublicAddress, amount: u64);
```

Allows `spender` to withdraw from `calling_account` up to `amount`. If this method is called again, it overwrites the allowance with `amount`.

`set_allowance` must panic if `balance_of(calling_account)` < `amount`.

Log `SetAllowance` must be emitted if `set_allowance` is successful.

## Required Logs

In this section, `++` denotes bytes concatenation.

### `Transfer`

| Field | Value |
| ----- | ----- |
| Topic | `0u8` ++ `owner: PublicAddress` ++ `recipient: Option<PublicAddress>`  |
| Value | `amount (little-endian bytes from u64)` |

Gets emitted on a successful token transfer through methods `transfer` and `transfer_from`.

### `SetAllowance`

| Field | Value |
| ----- | ----- |
| Topic | `1u8` ++ `owner: PublicAddress` ++ `spender: PublicAddress` |
| Value | `amount (little-endian bytes from u64)` |

Gets triggered on successful delegation of tokens through method `set_allowance`. 
