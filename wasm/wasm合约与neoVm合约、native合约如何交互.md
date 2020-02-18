# Ontology Wasm合约与NeoVm合约以及native如何交互
目前，Ontology主网已经支持Wasm合约，到现在为止，Ontology链已经支持三种合约，第一种是native合约，也是Ontology的原生合约，直接由golang语言实现，在创世块中部署，执行速度快；第二种是NeoVm合约，这种合约的运行在NeoVm虚拟机上，具有合约文件小，字节码简单，性能高的特点;第三种是Wasm合约，这种合约支持多种高级语言开发的程序直接编程wasm字节码，功能更加丰富，可以直接引用很多优秀的第三方库，Wasm社区也比较活跃。那么Ontology链上的Wasm合约如何调用native合约和neovm合约呢？本文将会详细介绍该调用如何实现。
在介绍下面的调用过程中，大家可以先把我们的合约模板clone下来，然后修改`lib.rs`文件，进行测试。
## rumtime模块中跨合约调用接口介绍
ontology-wasm-cdt-rust库中封装了跨合约调用的一个通用接口，如下
```
pub fn call_contract(addr: &Address, input: &[u8]) -> Option<Vec<u8>>
```
该方法需要两个参数，第一个是`addr`目标合约地址，第二个是`input`调用的目标合约的方法名和方法参数。在跨合约调用的过程中，要按照正确的方式序列化方法名和方法参数，其中调用NeoVm合约和调用native合约序列化方法名和方法参数是不一样的，下面我们会详细介绍如何正确的序列化方法名和方法参数。

## Wasm合约调用native合约
ontology-wasm-cdt-rust库中封装了`ont`和`ong`合约调用接口，只需要通过`use ostd::contract::ont;`引入即可，调用比较简单，现仅列出`ont`转账的合约调用示例
```
use ostd::contract::ont;
...
let (from, to, amount) = source.read().unwrap();
sink.write(ont::transfer(from, to, amount));
```
现在，我们看一下`ont::transfer`方法的实现源码
```
pub fn transfer(from: &Address, to: &Address, val: U128) -> bool {
    let state = [TransferParam { from: *from, to: *to, amount: val }];
    super::util::transfer_inner(&ONT_CONTRACT_ADDRESS, state.as_ref())
}
```
由上面的源码可以看到，先是构造一个`TransferParam`类型的实例,然后构造一个数组，这一步了是为了支持一笔交易中具有多笔转账的功能。最后将`ont`的合约地址和构造好的数组引用传给`util::transfer_inner`方法，我们接着看`util::transfer_inner`的实现源码如下：
```
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
由之前的教程，我们知道，在合约中进行参数序列化的工具是`Sink`实例，所以，先构造一个`Sink`实例，由于要序列化的参数是个数组，所以要先序列化该数组的长度，在序列化数组长度的时候，要将数组长度要转换成`u64`的数据类型，然后调用`sink.write_native_varuint`方法进行序列化，数组长度序列化好后，开始序列化数组中的每个元素，对于`Address`类型的数据，需要调用`sink.write_native_address`方法进行序列化，对于`U128`类型的数据，需要先将其转换成bytearray类型，然后进行序列化，`U128`转换成bytearray类型的数据需要调用`u128_to_neo_bytes`方法，至此，我们将方法参数序列化好了，下面我们要开始序列化方法名，这个时候，需要重新构造一个序列化实例，用这个新的实例序列化方法名，在序列化方法名之前，先序列化`Version`，该字段默认是0，然后序列化方法名，最后再序列化刚才序列化好的方法参数。至此，调用native合约方法的参数构造过程已完成，可以调用`runtime`接口中的方法进行调用了。

## Wasm合约调用NeoVm合约
Wasm合约调用NeoVm合约时，要求传递的参数类型实现`VmValueEncoder`和`VmValueDecoder`接口，ontology-wasm-cdt-rust库已经为常用的数据类型实现了该接口，例如：&str、&[u8]、bool、H256、U128、Address。`contract`模块中封装好了`neo`模块，开发者可以直接使用`neo`模块来调用NeoVm合约，使用示例如下：
```
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

`neo::call_contract`需要两个参数，第一个是调用的目的合约地址，第二个是调用的合约方法需要的方法名和方法参数，在上面的例子中，方法名是`init`，方法参数是一个空的Tuple类型的数据。Wasm合约中调用Neo合约时，得到的返回值，需要使用`VmValueParser`进行反序列化，拿到合约返回结果。下面我们看一下`neo::call_contract`的实现源码，如下：
```
pub fn call_contract<T: crate::abi::VmValueEncoder>(
        contract_address: &Address, param: T,
) -> Option<Vec<u8>> {
    let mut builder = crate::abi::VmValueBuilder::new();
    param.serialize(&mut builder);
    crate::runtime::call_contract(contract_address, &builder.bytes())
}
```
从上面的源码可以看到`neo::call_contract`方法需要两个参数，第一个参数是目标合约地址，第二个就是方法名和方法参数，并且要求方法名和方法参数必须实现`VmValueEncoder`接口，在对方法名和方法参数序列的时候，用的是`VmValueBuilder`而不是`Sink`，这一点值得开发者注意。得益于rust强大的宏功能，我们使用宏为Tuple类型的数据实现`VmValueEncoder`和`VmValueDecoder`接口，所以在调用的时候，我们传进来的是Tuple类型的数据("inti",())。
