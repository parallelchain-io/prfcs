# ParallelChain Request for Comments 4 (PRFC 4)

| PRFC | Title | Author | Version | Date First Published |
| --- | ----- | ---- | --- | --- |
| 4   | Support Interface | ParallelChain Lab | 1 | Oct 24th, 2023 | 

## Summary
---

ParallelChain Request for Comments 4 defines a standard interface for checking if a contract implements certain interfaces. Similar to Ethereum's ERC 165, this proposal defines a common method to tell the contract caller about whether a contract supports a set of methods in an efficient way.

## Required View
---

The following uses syntax from Rust (version 1.59.0).

```rust
fn supports_interface(interface_hashes: Vec<[u8; 32]>) -> Vec<[u8; 32]>;
```

This method accepts a list of hashes calculated from some subsets of the **method signatures** in a contract. The subsets of method signatures includes the methods required to be implemented in order to support certain interface. It outputs a list of matching hashes as it is possible for a contract to implement more than one interfaces.


### Method Signature

The method signature is a concatenation of
- the method name (String). Case sensitive.
- a character "(".
- method argument types seperated by ",". The receiver (if any) is ignored. Empty String if no argument.
- a character ")".
- the return type. Empty String if no return type.


```rust
// Example 1
fn method(&self, i: u16) -> String; // the method signature is "method(u16)String"

// Example 2
fn hello(); // the method signature is "hello()"
```

The data type of input argument and return type is taken from the code directly. It means the name of the type alias is used if a data type is a type alias.

```rust
// For example

type MyType = String;

fn method() -> MyType; // the method signature is "method()MyType"

```

Note that the actual implementation of the method signature may vary between different contract designs. Contract providers may consider providing the computed method signatures to prevent conflicts.

### Interface Hash

The interface hash is computed by **byte-wise** XOR operation over all SHA256 hashes on method signature as UTF8 bytes.

Formula:
```rust
interface_hash = sha256(method_sig_1) ^ sha256(method_sig_2) ^ ... ^ sha256(method_sig_n);
```

Rust implementation for byte-wise XOR operation and the `interface_hash` function:
```rust
fn xor(a: [u8; 32], b: [u8; 32]) -> [u8; 32] {
    a.iter().zip(b)
    .map(|(a_i, b_i)| a_i ^ b_i)
    .collect::<Vec<u8>>()
    .try_into()
    .unwrap()
}

fn interface_hash(method_signatures: Vec<&str>) -> [u8; 32] {
    method_signatures.into_iter()
    .map(str::as_bytes)
    .map(sha256)
    .reduce(xor)
    .unwrap()
}
```
