# Ontology WASM contract runtime API specification
 
The **runtime** module of the `ontology-wasm-cdt-rust` Ontology WASM contract development toolkit includes APIs that enable communication between the contract and Ontology blockchain. The API methods can be used to fetch on-chain data and store the contract data on the chain. The API methods have been listed below:

| API Methods                    | Response Value | Description                                                                                           |
| :----------------------------- | :------------: | :---------------------------------------------------------------------------------------------------- |
| timestamp                      |      u64       | Fetch current timestamp                                                                               |
| block_height                   |      u32       | Fetch current blockheight                                                                             |
| address                        |    Address     | Fetch address of the contract that is run                                                             |
| caller                         |    Address     | Fetch the address of the party invoking the contract, mainly used in certain cross contract scenarios |
| entry_address                  |    Address     | Fetch the entry address                                                                               |
| current_blockhash              |      H256      | Fetch current block's hash                                                                            |
| current_txhash                 |      H256      | Fetch curent transaction's hash                                                                       |
| sha256(data: impl AsRef<[u8]>) |      H256      | Calculate the SHA256 encryption of the input parameter                                                |
| check_witness(addr: &Address)  |      bool      | Check whether the specified address's signature exists                                                |
| input                          |    Vec<u8>     | Fetch the parameters passed when the contract was invoked                                             |
| ret(data: &[u8])               |       !        | Returns the result of contract execution                                                              |
| notify(data: &[u8])            |                | Save the contract's `notify` content on the blockchain                                                |
| panic(msg: &str)               |       !        | Contract's `panic` message                                                                            |

Next, we will describe the available API methods in detail. Developers are advised to first clone our smart contract template from Github and then add the contract logic in `lib.rs` file.

## Runtime API Usage Method

Developers can use the following command to import the `runtime` module into the contract.

```rust
use ontio_std::runtime;
```

All the API methods can be called using the `runtime` module. The available methods are:

1. timestamp()

The `timestamp()` method can be used to fetch the current timestamp. The value returns the UNIX timestamp in seconds. Example:

```rust
let t = runtime::timestamp();
```
1. block_height()

The `block_height` method can be used to fetch the current height of the blockchain. Example:

```rust
let t = runtime::block_height();
```

1. address()

The `address()` method can be used to fetch a contract's address. Example:

```rust
let t = runtime::address();
```

1. caller()

The `caller()` method can be used to fetch the address of the party calling the contract. This finds application in cross-contract scenarios, for example if a contract A calls another contract B, contract B can use this method to find out the address of contract A. Example:

```rust
let t = runtime::caller();
```
1. entry_address()

The `entry_address()` can be used to fetch the entry address of a contract. A sample application could be where a contract A calls a contract C through contract B, and the contract C uses this method to fetch the address of contract A. Example:

```rust
let t = runtime::entry_address();
```
6 current_blockhash()

The `current_blockhash()` method can be used to fetch the hash of the current block. Example:

```rust
let t = runtime::current_blockhash();
```
1. current_txhash()

The `current_txhash()` method can be used to fetch the hash of the current transaction. Example:

```rust
let t = runtime::current_txhash();
```
1. sha256()

This `sha256()` method can be used to fetch the **SHA256** encryption of the input parameter. Example:

```rust
let h = runtime::sha256("test");
```
1. check_witness()

The `check_witness(from)` verifies wheter the signature of the passed address exists.

* The method checks whether the party invoking the method contains the signauture of `from`. If true (and signature verification is successful), the method returns `true`.
* The method checks whether the invoker is a contract. If it is, and the method is invoked from this contract, it returns `true`. It also checks if the `from` is the returned value from `caller()`. Here, the `caller()` method returns the contract hash of the contract that invokes the method.
  
```rust
assert!(runtime::check_witness(from));
```

1.   notify()

The `notify` method can be used to pass contract event information to the network along with transmitting it to the blockchain. Example:

```rust
runtime::notify("notify".as_bytes())
```
An event function can be defined when sending a message from the contract using the `#[event]` annotation. The toolkit provided includes the necessary macros which can be imported using `use ostd::macros::event;`. Example:

``` rust
use ostd::macros::event;
mod notify {
    use super::*;
    #[event]
    pub fn transfer(from: &Address, to: &Address, amount: U128) {}
}
fn transfer(from: &Address, to: &Address, amount: U128) -> bool {
	...
	notify::transfer(from, to, amount);
}
```

1.  panic()

The panic method stops a transaction when a critical error occurs and then rolls back the current transaction.
This method can prove to be very useful in a cross-contract scanario. For example, before contract A's method calls contract B's method, it trasmits and stores certain data to the blockchain, but before contract B's method can be executed a critical error occurs. At his point, the action performed by contract A and the data stored on the chain need to be rolled back. This is carried out by using the `panic` function in contract B's method. Example:

```rust
runtime::panic("test");
```
