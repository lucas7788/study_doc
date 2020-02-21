# Ontology WASM, NeoVM, and native smart contract interaction

# Ontology WASM

Ontology mainnet currently supports three kinds of smart contracts-
1. **Native contract**: The contract native to Ontology system, implemented in Golang and deployed in the Genesis block. Native contracts offer quick execution times.
2. **NeoVM contract**: A NeoVM contract is run on the NeoVM engine. Certain charactersitics of NeoVM smart contract are small contract file size, simple bytecode, and high performance.
3. **WASM contract**: WASM contracts support multiple high level languages that can be compiled to bytecode. WASM contracts provide rich functionality, natively support several third-party database. The WASM development community is also very active.

But how do WASM contracts invoke native and NeoVM contracts? Here we illustrate how the mechanism is implemented.

Developers can clone the contract template, edit the `lib.rs` file, and start testing.

## Cross Contract API Call Using the Runtime Module

A general API has been encapsulated in the `ontology-wasm-cdt-rust` library, which can be used as follows:

```rust
pub fn call_contract(addr: &Address, input: &[u8]) -> Option<Vec<u8>>
```

This method takes two parameters. The `addr` parameter indicates the target contract address, and the `input` parameter is the name of the method to be invoked from the target contract and it's parameters. The name and parameters of the function should be correctly serialized. There are a few differences between the serialization process for NeoVM and native contract method and parameters. The details regarding serialization will be specified below.

## WASM Contract Invokes a Native Contract

The `ontology-wasm-cdt-rust` library includes API that can be used to invoke the `ONT` and `ONG` contracts. The `use ostd::contract::ont;` declaration can be used to import it and use it conveniently. An example of implementing an `ONT` transfer can be referred to below:

```rust
use ostd::contract::ont;
...
let (from, to, amount) = source.read().unwrap();
sink.write(ont::transfer(from, to, amount));
```

The source code for `ont::transfer` method is as follows:

```rust
pub fn transfer(from: &Address, to: &Address, val: U128) -> bool {
    let state = [TransferParam { from: *from, to: *to, amount: val }];
    super::util::transfer_inner(&ONT_CONTRACT_ADDRESS, state.as_ref())
}
```

The code above clearly illustrates that first an instance of `TransferParam` type is created, and then an array is defined. This is done to support multi account transfer. Next, the `ONT` contract address and the array created are passed to `util::transfer_inner` method. The definition for the `util::transfer_inner` method is as follows:

```rust
pub(crate) fn transfer_inner(
    contract_address: &Address, transfer: &[super::TransferParam],
) -> bool {
    let mut sink = Sink::new(64);
    sink.write_native_varuint(transfer.len() as u64);

    for state in transfer.iter() {
        sink.write_native_address(&state.from);
        sink.write_native_address(&state.to);
        sink.write(u128_to_neo_bytes(state.amount));
    }
    let mut sink_param = Sink::new(64);
    sink_param.write(VERSION);
    sink_param.write("transfer");
    sink_param.write(sink.bytes());
    let res = runtime::call_contract(contract_address, sink_param.bytes());
    if let Some(data) = res {
        if !data.is_empty() {
            return true;
        }
    }
    false
}
```

The sample code above clearly illustrates that the tool used to serialize the parameters is a `Sink` instance. Since the parameter to be serialized is an array, the array length is serialized, and the type is converted to `U64` array before invoking the `sink.write_native_varuint` method to carry out serialization. Each element of the array is serialized after the array length serialized.
The address is serialized using the `sink.write_native_address`. The data of `U128` type is first converted to `bytearray` and then serialized. This conversion can be carried out using the `u128_to_neo_bytes` method. At this point, the parameters have been serialized.
To serialize the method name we first need to create a serialization instance that will be used to serialize the method name. Before the method name is serialized, the `version` is serialized first. This field is set to `0` by default. Next the method name is serialized and parameters are serialized again. Here, the conditions to invoke a native contract have been fulfilled and the `runtime` APIs method can be used to invoke the contract.

## WASM Contract Invokes a NeoVM Contract

When a NeoVM contract is invoked by a WASM contract, the `VmValueEncoder` and the `VmValueDecoder` API can be implemented to transfer the parameters. The `ontology-wasm-cdt-rust` library supports most commonly used data types, for e.g. `&str`, `&[u8]`, `bool`, `H256`, `U128`, `Address`. The `contract` module encasuplates the `neo` module and allows developers to use it's corresponding methods to invoke NeoVM contract. Refer to the sample code below:

```rust
use ostd::contract::neo;
...
let res = neo::call_contract(&NEO_CONTRACT_ADDR, ("init", ()));
match res {
    Some(res2) => {
        let mut parser = VmValueParser::new(res2.as_slice());
        let r = parser.bool();
        sink.write(r.unwrap_or(false));
    }
    _ => sink.write(false),
}
```

The `neo::call_contract` method takes two parameters. The first parameter is the target contract's address, and the second parameter is the name and parameters of the method to be called from the target contract.
In the sample code above the method name is `init`, and the parameter passed is an empty tuple. The return value from the method needs to be serialized using the `VmValueParser` to obtain the final result.

The `neo::call_contract` method is defined as follows:

```rust
pub fn call_contract<T: crate::abi::VmValueEncoder>(
        contract_address: &Address, param: T,
) -> Option<Vec<u8>> {
    let mut builder = crate::abi::VmValueBuilder::new();
    param.serialize(&mut builder);
    crate::runtime::call_contract(contract_address, &builder.bytes())
}
```

The code above shows that the `neo::call_contract` takes two parameters. The first parameter is the target contract address and the second parameter is the method name and and the required parameters. The method name and the parameters must implement the `VmValueEncoder` API. 

> The `VmValueBuilder` method should be used to serialize method name and the parameters instead of using `Sink`. 

The macro function is a powerful feature of the Rust programming language. Macros are used to implement `VmValueEncoder` and `VmValueDecoder` for tuple type data. Tuple data `("inti",())` is imported when invoking the method.