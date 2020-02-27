# Wasm合约开发基础

1. Wasm合约开发模板
为了方便开发者使用Rust语言开发Ontology Wasm合约，我们提供了合约模板，该模板提供了合约编译脚本以及合约编写基本的文件结构，[git下载地址](https://github.com/ontio/rust-wasm-contract-template)

2. Processing Contract Invocation Parameters

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

3. Wasm如何调用Ont和Ong转账
Ontology Wasm合约开发工具集已经封装好了调用Ont和Ong合约的方法，Ont和ong合约调用相关方法都在`contract`模块里面，
调用示例如下
```
#![no_std]
use ostd::contract::ont;
fn ont_transfer(&self, from: &Address, to: &Address, amount: U128) -> bool {
    ont::transfer(&from, &to, amount)
}
fn ong_transfer(&self, from: &Address, to: &Address, amount: U128) -> bool {
    ong::transfer(&from, &to, amount)
}
#[no_mangle]
pub fn invoke() {
    let input = runtime::input();
    let mut source = Source::new(&input);
    let action = source.read().unwrap();
    let mut sink = Sink::new(12);
    match action {
        "ont_transfer" => {
            let (from, to, amount) = source.read().unwrap();
            sink.write(ont_transfer(from, to, amount));
          },
          "ong_transfer" => {
              let (from, to, amount) = source.read().unwrap();
              sink.write(ong_transfer(from, to, amount));
            },
        _ => panic!("unsupported action!"),
    }

    runtime::ret(sink.bytes())
}
```

4. Serializing the User-defined Data Structures


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

5. Fetching On-chain Data of Specific Data Types

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
