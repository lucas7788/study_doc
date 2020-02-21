# Data Serialization and Deserialization of Data in Ontology WASM Contract

Some common aspects of the development process are:
1. Parameter processing upon contract invocation
2. Save the self-defined structures on the blockchain
3. Fetch and read the on-chain data and process and resolve them to original data types
4. Transferring the parameters to the target contract when carrying out cross-contract invocation

Most of these aspects involve serialization and deserialization of data. Here we talk about data serialization and deserialization in Ontology WASM contracts.

## Encoder and Decoder Interfaces

The Encoder and Decoder interfaces define serialization and deserialization methods respectively for different data types. For the specific implementation logic, please refer to the `sink.rs` file. The `ontology-wasm-cdt-rust` library supports most commonly used data types, for e.g. `&str`, `u8`, `u16`, `u32`, `u64`, `u128`, `bool`, `H256`, `Address`, `Vec`, `String`, and `tuple`. Sample code for serialization and deserialization of different data types:


```rust
let mut sink = Sink::new(16);
sink.write(1u128);
sink.write(true);
let addr = Address::repeat_byte(1);
sink.write(addr);
sink.write("test");
let vec_data = vec!["hello","world"];
sink.write(&vec_data);
let tuple_data = (1u8, "hello");
sink.write(tuple_data);
let mut source = Source::new(sink.bytes());
let res1:u128 = source.read().unwrap_or_default();
let res2:bool = source.read().unwrap_or_default();
let res3:Address = source.read().unwrap_or_default();
let res4:&str = source.read().unwrap_or_default();
let res5:Vec<&str> = source.read().unwrap_or_default();
let res6:(u8, &str) = source.read().unwrap_or_default();
assert_eq!(res1, 1u128);
assert_eq!(res2, true);
assert_eq!(res3, addr);
assert_eq!(res4, "test");
assert_eq!(res5, vec_data);
assert_eq!(res6, tuple_data);
```

All the parameters passed to the `sink.write()` method support all the data types that implement the Encoder interface. Their data types need to be declared when serializing, for example `1u128`.
THe `source.read()` method can fetch and read all the data types that implement the Decoder interface. The target data type needs to be specified when deserializing as follows:

```rust
let res1:u8 = source.read().unwrap_or_default();
```

The data type needs to be specified after `res1`. Here, the data type is `u8`.

```rust
let res1 = source.read_byte().unwrap_or_default();
```

Since reading is carried out using the the `read_byte` method it is not necessary to specify the data type.

## Processing Contract Invocation Parameters


The contract fetches the invocation parameters using the `runtime::input()` method. But, this method can only fetch parameters of the `bytearray` type. This parameter can be deserialized to the corresponding method name and method parameters. Sample code for serialization and deserialization:

```rust
let input = runtime::input();
let mut source = Source::new(&input);
let method_name: &str = source.read().unwrap();
let mut sink = Sink::new();
match method_name {
    "transfer" => {
        let (from, to, amount) = source.read().unwrap();
        sink.write(ont::transfer(from, to, amount));
    }
    _ => panic!("unsupported action!"),
}
```

Rust supports type derivation. For most cases, type declaration is not necessary. For example, `ont::transfer()` method decalares the data type for `from`, `to`, and `amount` variables, and so data type declaration  is not required for these variables.

Deserialization for the `Vec<&str>` data type can be carried out in the following way:

```rust
let param:Vec<&str> = source.read().unwrap();
```

Deserialization for the `Vec<(&str, U128, Address)>` data type can be carried out in the following way:

```rust
let param:Vec<(&str,U128,Address)>= source.read().unwrap();
```

## Serializing the User-defined Data Structures


While developing smart contracts we often need to serialize and deserialize data of type `struct`. To achieve this function, `#[derive(Encoder, Decoder)]` can be conveniniently added above the `struct` declaration statement. Refer to the code:

```rust
#[derive(Encoder, Decoder)]
struct ReceiveRecord {
    account: Address,
    amount: u64,
}

#[derive(Encoder, Decoder)]
struct EnvlopeStruct {
    token_addr: Address,
    total_amount: u64,
    total_package_count: u64,
    remain_amount: u64,
    remain_package_count: u64,
    records: Vec<ReceiveRecord>,
}
```

> When using this feature to carry out serialization and deserialization, it is important to ensure that all the fields of `struct` implement the `Encoder` and `Decoder` interface.

```rust
let addr = Address::repeat_byte(1);
let rr = ReceiveRecord{
    account: addr,
    amount: 1u64,
};
let es = EnvlopeStruct{
    token_addr: addr,
    total_amount: 1u64,
    total_package_count: 1u64,
    remain_amount: 1u64,
    remain_package_count: 1u64,
    records: vec![rr],
};
let mut sink = Sink::new(16);
sink.write(&es);

let mut source = Source::new(sink.bytes());
let es2:EnvlopeStruct = source.read().unwrap();

assert_eq!(&es.token_addr,&es2.token_addr);
assert_eq!(&es.total_amount,&es2.total_amount);
assert_eq!(&es.total_package_count,&es2.total_package_count);
assert_eq!(&es.remain_amount,&es2.remain_amount);
assert_eq!(&es.remain_package_count,&es2.remain_package_count);
```

## Fetching On-chain Data of Specific Data Types

The different types of data that are handled in a contract can be stored on the blockchain. But these different data types first need to be converted to `bytearray` type using serialization. Similarly, the data that is fetched from the blockchain is in the `bytearray` form and needs to be deserialized to obtain results in specific data types.
The `database` module provides several simple API methods that are available for the developer to use.

```rust
put<K: AsRef<[u8]>, T: Encoder>(key: K, val: T)
```

The `T` data type is saved based on the `key`, and the type `T` is to implement the Encoder interface.

Sample code:

```rust
let es = EnvlopeStruct{
    token_addr: addr,
    total_amount: 1u64,
    total_package_count: 1u64,
    remain_amount: 1u64,
    remain_package_count: 1u64,
    records: vec![rr],
};
database::put("key", es);
```

From the above example it is clear that when using the `database::put` method, first we seialize the `es` parameter, and then save the result on the blockchain.

```rust
fn get<K: AsRef<[u8]>, T>(key: K) -> Option<T> where for<'a> T: Decoder<'a> + 'static,
```

Here we fetch data of type `T` using the `key`. The `T` data type requires implementation of the Decoder interface.

Example:

```rust
let res:EnvelopeStruct = database::get("key").unwrap();
```

From the example above we can see that the `database::get` method fetches data of the type `bytearray` from the chain, and then needs to be deserialized to obatin the result in the `EnvelopeStruct` format.

## Cross Contract Parameter Transfer

In the case of cross contract invocation, the parameters are transferred in the `bytearray` format. Thus, it is necessary to serialize different data types to `bytearray` format. An example for a cross contract invocation in the case of a WASM contract:

```rust
let (contract_address,method,a, b): (&Address,&str,u128, u128) = source.read().unwrap();
let mut sink = Sink::new(16);
sink.write(method);
sink.write(a);
sink.write(b);
let resv = runtime::call_contract(contract_address, sink.bytes()).expect("get no return");
```
