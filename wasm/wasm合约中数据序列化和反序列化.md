# Ontology Wasm合约中数据序列化和反序列化教程

我们在合约开发的过程中，总是会遇到以下情况：
1. 解析调用合约的参数
2. 将自定义的结构体保存到链上
3. 从链上读取已存在的数据并解析成原始的数据类型
4. 跨合约调用的时候，传递目的合约需要的参数。
上面所列出来的情况，均涉及到参数的序列化和反序列化问题，本文将会详细介绍Ontology wasm合约中参数序列化和反序列化的方法。

## Encoder和Decoder接口
Encoder接口定义了不同类型数据的序列化方法，具体实现逻辑请看`sink.rs`文件，Decoder接口定义了不同类型数据的反序列化方法，具体实现逻辑代码请看`source.rs`文件。ontology-wasm-cdt-rust工具库中已经为常用的数据类型实现Encoder和Decoder接口，有u8、u16、u32、u64、u128、bool、Address、H256、Vec、&str、String和Tuple类型。
常用数据类型的序列化和反序列化示例如下：
```
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
`sink.write()`方法传入的参数支持所有已经实现Encoder接口的数据类型,序列化的时候需要声明其数据类型,比如1u128等。
`source.read()`方法可以读取所有已经实现Decoder接口的数据类型，反序列化的时候要指明其结果的数据类型，如下面的代码
```
let res1:u8 = source.read().unwrap_or_default();
```
在res1后面要声明其数据类型是u8,也可以写成
```
let res1 = source.read_byte().unwrap_or_default();
```
在读取的时候，已经使用了`read_byte`所以不需要在res1后面声明数据类型了。

## 解析调用合约的参数

合约在获得调用参数的时候是通过`runtime::input()`方法获得的，但是该方法仅能拿到`bytearray`格式的参数，需要反序列化成对应的参数，第一个参数是合约方法名，后面的是方法参数，示例如下：
```
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
rust支持类型推导，大部分情况下可以省略类型声明，比如在`ont::transfer()`方法中已经声明了from,to,amount的数据类型，所以在前面解析from、to、amount的时候没有声明数据类型。

如果参数是Vec<&str>类型，可以按照如下的方式进行反序列化
```
let param:Vec<&str> = source.read().unwrap();
```
如果参数是Vec<(&str,U128,Address)>类型，可以按照如下的方式进行反序列化
```
let param:Vec<(&str,U128,Address)>= source.read().unwrap();
```

## 序列化自定义结构体
在合约开发中，我们经常需要对`struct`类型的数据进行序列化和反序列化，为了达到这个目的，我们仅需要在`struct`声明的地方加上`#[derive(Encoder, Decoder)]`即可，示例如下：
```
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
在使用该功能的时候，要注意`struct`的每个字段必须都已经实现`Encoder`和`Decoder`接口，加上该注解后，就可以按照如下的方式序列化和反序列化了：
```
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
## 从链上读取指定类型的数据
合约中的不同类型的数据在保存到链上之前，需要先序列化成`bytearray`类型的数据，从链上读取数据时，读到的也都是`bytearray`类型的数据，需要反序列化成指定的数据类型。
`database`模块提供了更加简便的接口供开发者使用
* put<K: AsRef<[u8]>, T: Encoder>(key: K, val: T)
根据key保存T类型的数据，要求T类型实现`Encoder`接口
示例:
```
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
我们从`database::put`的源码可以看到，该方法在执行的时候，会先序列化es参数，然后将序列化结果保存到链上。
* fn get<K: AsRef<[u8]>, T>(key: K) -> Option<T> where for<'a> T: Decoder<'a> + 'static,
根据key获得指定类型T的数据,T类型要求实现了Decoder接口。
示例:
```
let res:EnvlopeStruct = database::get("key").unwrap();
```
我们从`database::get`的源码可以看到，该方法在执行的时候，会先从链上拿到bytearray类型的数据，然后再反序列化得到EnvlopeStruct类型的数据。

## 跨合约参数传递
在跨合约调用的时候，参数是以bytearray的格式进行传递的，所以需要将不同类型的数据进行序列化,下面以跨合约调用wasm合约为例。
```
let (contract_address,method,a, b): (&Address,&str,u128, u128) = source.read().unwrap();
let mut sink = Sink::new(16);
sink.write(method);
sink.write(a);
sink.write(b);
let resv = runtime::call_contract(contract_address, sink.bytes()).expect("get no return");
```
