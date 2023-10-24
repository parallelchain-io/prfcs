# ParallelChain Request for Comments 2 (PRFC 2)

| PRFC | Title | Author | Version | Date First Published |
| --- | ----- | ---- | --- | --- |
| 2   | Non-Fungible Token Standard | ParallelChain Lab | 3 | May 10th, 2023 | 

## Summary
---
  
ParallelChain Request for Comments 2 defines a standard interface for non-fungible tokens implemented as ParallelChain smart contracts. "Non-Fungible Tokens" or NFTs is taken here to have the same meaning as in Ethereum's ERC-721, namely a set of transferable entities on a blockchain with identification metadata unique to its creator. For example, a deed for a plot of land is a non-fungible token, since a deed identifies a singular, unique plot of land (i.e., no two deeds identifies the same plot of land).

A standard contract interface for non-fungible tokens allows more seamless interoperability, since applications can make the simplifying assumption that all PRFC 2-implementing contracts always export the same, named set of Methods (they may export more).

The below sections list the set of methods that all smart contracts that want to be PRFC 2-compliant must implement, as well as the behavior that each defined method must exhibit. Required behavior involves emitting certain events. These are also listed and described.

## Glossary
---

**Collection**: A set of Tokens with shared properties. A single PRFC 2-implementing contract represents a single Collection. For example, 'CryptoKitties' is a Collection, whilst a single CryptoKitty is a Token.  

**Token**: An instance of a Collection.

**Owner**: An account that owns the token.

**Spender**: An account that is approved (using method `set_spender` or `set_exclusive_spender`) to transfer tokens on behalf of the owner. All Tokens can have at most one Spender. Spender is originally set to be the owner if not specified.

**Exclusive Spender**: An exclusive spender is an account that is approved to transfer *all* tokens owned by the owner. An account can be made an Spender of all tokens through a single call to `set_exclusive_spender`.

## Notes

- The following uses syntax from Rust (version 1.59.0).
- The data type `PublicAddress` is the type alias to a 32-byte slice `[u8; 32]`. 
- The term `calling_account` refers to the account that invokes the method.

## Required Types
---

Token Identifier is a String.

```rust
type TokenID = String;
```

Collection is borsh-serializable structure. It is essentially all information about the contract.

```rust
struct Collection {
    name: String,
    symbol: String,
    tokens: Vec<Token>,
    uri: Option<String>
} 
```

Token is borsh-serializable structure.

```rust
struct Token {
    id: TokenID,
    name: String,
    // Recommendation: uri should be an Internet URL, viewable on a browser.
    uri: String,
    owner: PublicAddress,
    spender: Option<PublicAddress>,
}
```

## Required Views 
---

### collection

```rust
fn collection() -> Collection
```

Returns information about the Collection represented by this contract.

### token_owned

```rust
fn token_owned(owner: PublicAddress) -> Vec<Token>
```

Returns *all* tokens owned by `owner`.

### tokens

```rust
fn tokens(ids: Vec<TokenID>) -> Vec<Token>
```

Returns information about the tokens identified by `ids`. 

If an ID does not identify a token, it must not appear in the returned vector. 

### owner

```rust
fn owner(id: TokenID) -> PublicAddress
```

Returns public address of the token owner identified by `id`.

### spender

```rust
fn spender(id: TokenID) -> Option<PublicAddress>
```

Returns public address of the token spender identified by `id`.

Returns `None` if spender is not specified.


## Required Calls
--- 

### transfer

```rust
fn transfer(token_id: TokenID, to_address: Option<PublicAddress>)
```

Transfers the token identified by `token_id` from the owner account identified by `calling_account`, to the account identified by `to_address`. If `to_address` is None, the token will be burnt.

`transfer` must panic if:
1. `calling_account` != `owner(token_id)`.
2. Or, if evaluating (1.) causes a panic.

Log `Transfer` must be triggered if `transfer` is successful.

### transfer_from

```rust
fn transfer_from(from_address: PublicAddress, to_address: Option<PublicAddress>, token_id: TokenID)
```
Transfers the token identified by `token_id` to the account identified by `to_address` on behalf of the token owner identified by `from_address`. If `to_address` is None, the token will be burnt.

`transfer_from` must panic if: 
1. Some(`calling_account`) != `spender(token_id)`.
2. `from_address` != `owner(token_id)`.
3. Or, if evaluating (1.) or (2.) causes a panic.

Log `Transfer` must be triggered if `transfer_from` is successful. 

### set_spender

```rust
fn set_spender(token_id: TokenID, spender: PublicAddress)
```

Grants the account identified by `spender` the right to transfer the token identified by `token_id` on behalf of its owner.

`set_spender` must panic if:
1. `calling_account` != `owner(token_id)`.
2. Or, if evaluating (1.) causes a panic.

Log `SetSpender` must be triggered if `set_spender` is successful.

### set_exclusive_spender

```rust
fn set_exclusive_spender(spender: PublicAddress)
```

Gives the account identified by `spender` the right to transfer *all* tokens owned by `calling_account`. Calling this method MUST have the same effects as calling `set_spender` for every token owned by `calling_account` with the same `spender`.

`set_exclusive_spender` must panic if:
1. `calling_account` != `owner(token_id)`.
2. Or, if evaluating (1.) causes a panic.

Log `SetExclusiveSpender` must be triggered if `set_exclusive_spender` is successful.
     
## Required Logs
---

In this section, `++` denotes bytes concatenation.

### `Transfer`

| Field | Value |
| ----- | ----- |
| Topic | `0u8` ++ `token_id: UTF8 Bytes` ++ `owner: PublicAddress` ++ `recipient: Option<PublicAddress>` |
| Value | `0u8` |

Gets triggered on successful call to methods `transfer`, or `transfer_from`.

### `SetSpender`

| Field | Value |
| ----- | ----- |
| Topic | `1u8` ++ `token_id: UTF8 Bytes` ++ `spender: PublicAddress` |
| Value | `1u8` |

Gets triggered on successful call to method `set_spender`.

### `SetExclusiveSpender`

| Field | Value |
| ----- | ----- |
| Topic | `2u8` ++ `owner: PublicAddress` ++ `spender: PublicAddress` |
| Value | `2u8` |

Gets triggered on successful call to method `set_exclusive_spender`. 
