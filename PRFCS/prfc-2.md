# ParallelChain Request for Comments 2 (PRFC 2)

| PRFC | Title | Author | Version | Date First Published |
| --- | ----- | ---- | --- | --- |
| 2   | Non-Fungible Token Standard | ParallelChain Lab | 4 (WIP) | 22 January, 2024 (WIP) | 

## Summary
  
The Non-Fungible Token Standard (PRFC 2) defines a standard interface for non-fungible tokens implemented as ParallelChain smart contracts. "Non-Fungible Tokens" or NFTs is taken here to have the same meaning as in Ethereum's ERC-721, namely a set of transferable entities on a blockchain with identification metadata unique to its creator. For example, a title for a plot of land is a non-fungible token, since a title identifies a singular, unique plot of land (i.e., no two titles identifies the same plot of land).

A standard contract interface for non-fungible tokens allows more seamless interoperability, since applications can make the simplifying assumption that all PRFC 2-implementing contracts always export the same, named set of Methods (they may export more).

The below sections list the set of methods that all smart contracts that want to be PRFC 2-compliant must implement, as well as the behavior that each defined method must exhibit. Required behavior involves emitting certain log messages. These are also listed and described.

## Entities

PRFC 2 contracts implement two kinds of entities: Tokens, and Collections.

### Token

A Token is an entity that represents a transferable, non-fungible (or, "unique") object, for example a deed for a plot of land. A single PRFC 2 contract could define multiple tokens, each identified by a String called a Token ID (`TokenID`). 

Within a single contract, token IDs are *unique*. That is, any specific token ID can be associated with *at most* one token in a single PRFC 2 contract, but the same token ID can be used to refer to two distinct tokens in two different PRFC 2 contracts.

#### Token Type

Tokens are modelled inside PRFC 2 contracts as a struct with five fields:

```rust
struct Token {
    id: TokenID,
    name: String,
    // Recommendation: uri should be an Internet URL, viewable on a browser.
    uri: String,
    owner: PublicAddress,
    spender: Option<PublicAddress>,
}

type TokenID = String;
```

### Collection

A Collection is a set of tokens that share common attributes in some business domain. A single PRFC 2 contract represents a *single* Collection. For example, a PRFC 2 contract could store a collection of land titles, with each token in the collection referring to a title for a different plot of land.

#### Collection Type

Collections are modelled inside PRFC 2 contracts as a struct with four fields:

```rust
struct Collection {
    name: String,
    symbol: String,
    tokens: Vec<Token>,
    uri: Option<String>
} 
```

## User Roles

Users of PRFC 2 contracts are identified by their ParallelChain [account](https://github.com/parallelchain-io/parallelchain-protocol/blob/master/World%20State.md#account), and thus by their unique `PublicAddress`. Further, users of PRFC 2 contracts are associated with a set of privileges called User Roles.

PRFC 2 defines 3 distinct user roles, namely Owner, Operator, and Spender. Each role as well as the relationships between the 3 roles are described in the following subsections.

### Owner

The owner role represents the privileges of an owner of a token. The owner of a token has the largest set of privileges with respect to that token out of the three roles. 

#### Uniqueness

Each token in a collection has exactly one owner account. Conversely, each account can be the owner of arbitrarily many tokens in a collection.

#### Privileges

The owner of a token is allowed to:
1. Transfer ownership of the token to a different account.
2. Designate a different account as the [spender] of the token.

Related to the above two privileges, *any* account can also:
1. Delegate "management" of all tokens it owns in the collection to a different account by giving it the [operator](#operator) role.

### Operator

The operator role represents the privileges of an account that has been designated by an owner account to “operate” or “manage” all tokens owned by the owner account in the specific collection.

#### Uniqueness

Within a single PRFC 2 contract, an owner account can have at most one operator. Additionally, an owner may not designate itself as its own operator account. 

#### Privileges

The operator of an owner account is allowed to:
1. Transfer ownership of any token owned by the owner account to a different account (including to itself).

### Spender

The spender role represents the privileges of an account that has been designated by the owner of a token to “manage” a specific token owned by the owner account (this is in contrast to the operator of the owner account, which can manage all tokens owned by the owner account).

#### Uniqueness

Within a single PRFC 2 contract, a single token can have at most one spender. Additionally, an owner may not designate itself as the spender of a token it owns.

Even though a single token can have at most one spender within a single a PRFC 2 contract, the roles of operator and spender are not absolutely mutually exclusive. In particular, an owner account A is allowed to designate an account B to be its operator, and then designate a different account C to be the spender of one of its tokens. Account B may be set as both the operator of account A, and a spender of one of A’s token, but this is essentially redundant.

#### Privileges

The spender of a token is allowed to:
1. Transfer ownership of the token to a different account (including to itself).

## Required View Methods

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


## Required Non-View Methods

### transfer

```rust
fn transfer(token_id: TokenID, to_address: Option<PublicAddress>)
```

Transfers the token identified by `token_id` from the `calling_account`, to the account identified by `to_address`. If `to_address` is None, the token will be burnt.

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

Grants the account identified by `spender` the right to transfer *all* tokens owned by the `calling_account`. Calling this method MUST have the same effects as calling `set_spender` for every token owned by `calling_account` with the same `spender`.

`set_exclusive_spender` must panic if:
1. `calling_account` != `owner(token_id)`.
2. Or, if evaluating (1.) causes a panic.

Log `SetExclusiveSpender` must be triggered if `set_exclusive_spender` is successful.
     
## Required Logs

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
