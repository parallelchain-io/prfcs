# PRFC 2: Non-Fungible Token Standard

| PRFC | Title | Author | Version | Date First Published |
| --- | ----- | ---- | --- | --- |
| 2   | Non-Fungible Token Standard | ParallelChain Lab | 4 | 22 January, 2024 | 

## Introduction
  
The Non-Fungible Token Standard (PRFC 2) defines a standard interface for non-fungible tokens implemented as ParallelChain smart contracts. "Non-Fungible Tokens" or NFTs is taken here to have the same meaning as in Ethereum's ERC-721, namely a set of transferable entities on a blockchain with identification metadata unique to its creator. For example, a title for a plot of land is a non-fungible token, since a title identifies a singular, unique plot of land (i.e., no two titles identifies the same plot of land).

## Organization

This specification is organized into seven sections:
1. [Entities](#entities) describes the two major entities implemented by PRFC 2 contracts: Tokens, and Collections.
2. [User Roles](#user-roles) describes the three user roles implemented by PRFC 2 contracts: Owner, Operator, and Spender.
3. [Required View Methods](#required-view-methods) specifies the set of [view methods](https://github.com/parallelchain-io/parallelchain-protocol/blob/master/Contracts.md#view-calls) that PRFC 2 contracts *must* implement.
4. [Required World State-mutating Methods](#required-world-state-mutating-methods) specifies the set of world-state mutating methods that PRFC 2 contracts *must* implement.
5. [PRFC 2 Metadata Extension](#prfc-2-metadata-extension) specifies a set of  *optional* view methods for getting metadata about tokens and collections.
6. [PRFC 2 Enumeration Extension](#prfc-2-enumeration-extension) specifies a set of *optional* view methods for enumerating through all tokens in a collection.
7. Finally, [Required Logs](#required-logs) specifies the set of logs that must be emitted by PRFC 2 contracts to notify users of specific, significant events.

## Entities

PRFC 2 contracts implement two kinds of entities: Tokens, and Collections. This section specifies each entity.

### Token

A Token is an entity that represents a transferable, non-fungible (or, "unique") object, for example a title for a plot of land. A single PRFC 2 contract could define multiple tokens, each identified by a String called a Token ID (`TokenID`). 

Within a single contract, token IDs are *unique*. That is, any specific token ID can be associated with *at most* one token in a single PRFC 2 contract, but the same token ID can be used to refer to two distinct tokens in two different PRFC 2 contracts.

### Collection

A Collection is a set of tokens that share common attributes in some business domain. A single PRFC 2 contract represents a *single* Collection. For example, a PRFC 2 contract could store a collection of land titles, with each token in the collection referring to a title for a different plot of land.

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
2. Designate a different account as the [spender](#spender) of the token.

Related to the above two privileges, *any* account can also:
1. Delegate "management" of all tokens it owns in the collection to multiple different accounts by granting them the [operator](#operator) role.

### Operator

The operator role represents the privileges of an account that has been designated by an owner account to “operate” or “manage” all tokens owned by the owner account in the specific collection.

#### Uniqueness

Within a single PRFC 2 contract, a single owner account can have *multiple* operators.

#### Privileges

The operator of an owner account is allowed to:
1. Transfer ownership of any token owned by the owner account to a different account (including to itself).
2. Designate an account other than the owner account as the [spender](#spender) of any of the owner's tokens.

### Spender

The spender role represents the privileges of an account that has been designated by the owner of a token to “manage” a specific token owned by the owner account (this is in contrast to the operator of the owner account, which can manage all tokens owned by the owner account).

#### Uniqueness

Within a single PRFC 2 contract, a single token can have at most one spender. 

Even though a single token can have at most one spender within a single a PRFC 2 contract, the roles of operator and spender are not absolutely mutually exclusive. In particular, an owner account A is allowed to designate an account B to be its operator, and then designate a different account C to be the spender of one of its tokens. Account B may be set as both the operator of account A, and a spender of one of A’s token, but this is essentially redundant.

#### Privileges

The spender of a token is allowed to:
1. Transfer ownership of the token to a different account (including to itself).

## Required View Methods

### Owner

```rust
fn owner(token_id: TokenID) -> PublicAddress
```

Returns the public address of the owner of the token identified by `token_id`. 

#### Panics

`owner` must panic if `token_id` does not identify a token.

### Operator

```rust
fn is_operator(owner: PublicAddress, operator: PublicAddress) -> bool
```

Check whether an account (identified by `operator`) is currently set as an operator for a given account (identified by `owner`).

### Spender

```rust
fn spender(token_id: TokenID) -> Option<PublicAddress>
```

Returns the public address of the spender of the token identified by `token_id` (if any).

## Required World State-mutating Methods

### Transfer From

```rust
fn transfer_from(from_address: PublicAddress, to_address: Option<PublicAddress>, token_id: TokenID)
```
Transfers the token identified by `token_id`, currently owned by the account identified by `from_address`, to the account identified by `to_address`.

#### Spender is always un-set on transfer

Upon a successful transfer, `transfer_from` must set the spender of the token identified by `token_id` to `None`, **bypassing** the checks in the [panics section](#panics-2) of `set_spender`

To clarify, the spender of the token identified by `token_id` is not allowed to set the spender of the token using `set_spender`, but is allowed to transfer the token to an account other than the owner, which sets the spender of the token to `None`.

#### Token burning

If `to_address` is `None`, the token will be destroyed. E.g., it must no longer be queryable from any of the [required view methods](#required-view-methods).

#### Panics

`transfer_from` must panic if:
1. The calling account (`calling_account()`) is not any of:
    - The owner of the token (`owner(token_id)`).
    - The operator of the owner of the token (`operator(owner(token_id))`).
    - The spender of the token (`spender(token_id)`).
2. The `from_address` is not the current owner of the token (`owner(token_id)`).
3. The `to_address` is already the current owner of the token (`owner(token_id)`).
4. Or if evaluating (1.) or (2.) or (3.) causes a panic.

`transfer_from` is taken to be successful if it does not panic.

#### Log

Log `TransferFrom` must be triggered if `transfer_from` is successful. 

### Set Spender

```rust
fn set_spender(token_id: TokenID, spender: Option<PublicAddress>)
```

Grant the account identified by `spender` the Spender role for the token identified by `token_id`.

#### Replacing the current spender

If `spender(token_id)` is currently Some, `set_spender` replaces the current spender.

#### Un-setting spender

If `spender` is `None`, `set_spender` removes the spender role from the current `spender(token_id)`.

#### Panics

`set_spender` must panic if:
1. The calling account (`calling_account()`) is not any of:
    - The owner of token (`owner(token_id)`).
    - The operator (`operator(owner(token_id))`) of the current owner of the token (`owner(token_id)`)
2. Of if evaluating (1.) causes a panic.

`set_spender` is taken to be successful if it does not panic.

#### Log

Log `SetSpender` must be triggered if `set_spender` is successful.

### Set Operator

```rust
fn set_operator(operator: PublicAddress, approved: bool)
```

If `approved`, grant the account identified by `operator` the operator role for the calling account. If `!approved`, revoke the operator role from `operator`.

#### Log

Log `SetOperator` must be triggered if `set_operator` is successful.

## PRFC 2 Metadata Extension

The PRFC 2 Metadata Extension is a set of methods that PRFC 2 contracts can optionally implement to enable users to get a richer set of information about the collection and the tokens in the collection.

### Collection Name

```rust
fn collection_name() -> String
```

Get the name of the collection.

### Collection Symbol

```rust
fn collection_symbol() -> String
```

Get the symbol of the collection.

### Token URI

```rust
fn token_uri(token_id: TokenID) -> String
```

Get a Uniform Resource Identifier (URI) pointing to metadata for a specific token (identified by `token_id`). This URI *should* point to a JSON object that implements the [Token Metadata JSON Schema](#token-metadata-json-schema).

#### Token Metadata JSON Schema

```rust
{
    "title": "Token Metadata",
    "type": "object",
    "properties": {
        "name": {
            "type": "string",
            "description": "Identifies the asset to which this NFT represents"
        },
        "description": {
            "type": "string",
            "description": "Describes the asset to which this NFT represents"
        },
        "image": {
            "type": "string",
            "description": "A URI pointing to a resource with mime type image/* representing the asset to which this NFT represents. Consider making any images at a width between 320 and 1080 pixels and aspect ratio between 1.91:1 and 4:5 inclusive."
        }
    }
}
```

## PRFC 2 Enumeration Extension 

The PRFC 2 Enumeration Extension is a set of methods that PRFC 2 contracts can optionally implement to helps users iterate through all of the tokens in the collection.

### Total Supply

```rust
fn total_supply() -> u64
```

Returns the number of tokens currently in the collection.

### Token by Index

```rust
fn token_by_index(index: u64) -> Option<TokenID>
```

Enumerates through every token in the collection using an index number. The sort order is not specified.

### Token of Owner by Index

```rust
fn token_of_owner_by_index(owner: PublicAddress, index: u64) -> Option<TokenID>
```

Enumerates through every token in the collection owned by the specified account. The sort order is not specified.

## Required Logs

In this section, "++" denotes bytes concatenation.

### Transfer From

| Field | Value |
| ----- | ----- |
| Topic | `0u8` ++ `token_id: Base64URL` ++ `from_address: PublicAddress` ++ `to_address: Option<PublicAddress>` |
| Value | Empty. |

Gets triggered on successful call to method `transfer_from`.

### Set Spender

| Field | Value |
| ----- | ----- |
| Topic | `1u8` ++ `token_id: Base64URL` ++ `spender: Option<PublicAddress>` |
| Value | Empty. |

Gets triggered on successful call to method `set_spender`.

### Set Operator

| Field | Value |
| ----- | ----- |
| Topic | `2u8` ++ `owner: PublicAddress` ++ `operator: PublicAddress` ++ `approved: bool` |
| Value | Empty. |

Gets triggered on successful call to method `set_operator`. 
