# Wasm合约如何与Ontology链交互

ontology wasm合约开发工具集ontology-wasm-cdt-rust里面runtime模块封装了合约与Ontology链交互的api,通过这些api，合约可以获得链上的数据，或者将合约中的数据保存到链上，下面列出了这些api的简单描述。
|API名称|返回值|描述|
|:--|:--|:--|
|timestamp|u64|获得当前的时间戳|
|block_height|u32|获得当前的区块高度|
|address|Address|获得当前执行的合约的地址|
|caller|Address|获得调用方的合约的地址，主要用于跨合约调用时，获得调用方的合约地址|
|entry_address|Address|获得入口合约地址|
|current_blockhash|H256|获得当前区块hash|
|current_txhash|H256|获得当前的交易hash|
|sha256(data: impl AsRef<[u8]>)|H256|计算输入参数的hash256值|
|check_witness(addr: &Address)|bool|校验是否含有该地址的签名|
|input|Vec<u8>|获得调用合约时的输入参数|
|ret(data: &[u8])|!|返回合约执行结果|
|notify(data: &[u8])||保存合约中notify到链上|
|panic(msg: &str)|!|合约中panic信息|

接下来，我们具体讲述这些Api的使用方法，在此之前，开发者可以从github上clone下来我们的合约模板，然后在`lib.rs`文件中添加合约逻辑代码。

## Runtime API 使用方法
开发者仅需要通过下面的方式将runtime模块引入到当前合约中。
```
use ontio_std::runtime;
```
然后就可以通过`runtime`引用以上所有的API接口。
1. notify API
Notify函数将合约中事件推送到全网，并将其内容保存到链上。调用方法如下：
```
#![no_std]
use ontio_std::runtime;

#[no_mangle]
fn invoke() {
	runtime::notify("notify".as_bytes())
}
```
2. timestamp() API
`timestamp()`方法获得当前的时间戳，即返回调用该函数的Unix时间，其单位是秒。
```
#![no_std]
use ontio_std::runtime;
use ontio_std::prelude::*;

#[no_mangle]
fn invoke() {
	let t = runtime::timestamp();
	runtime::ret(t.to_string().as_bytes());
}
```
