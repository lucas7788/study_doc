# Ontology Wasm合约runtime api教程
ontology wasm合约开发工具库ontology-wasm-cdt-rust里面runtime模块封装了合约与Ontology链交互的api,通过这些api，合约可以获得链上的数据，或者将合约中的数据保存到链上，下面列出了这些api的简单描述。
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

1. timestamp()

`timestamp()`方法获得当前的时间戳，即返回调用该函数的Unix时间，其单位是秒。
调用示例:
```
let t = runtime::timestamp();
```
2. block_height

`block_height`函数获得当前区块链网络的区块高度,调用示例：
```
let t = runtime::block_height();
```

3. address

`address` 获得当前合约的地址，调用示例：
```
let t = runtime::address();
```

4. caller

`caller` 获得调用方的合约地址，主要用于跨合约调用的场景,比如合约A调用合约B的应用场景, 在合约B中就可以调用该方法获得调用方合约A的地址：
```
let t = runtime::caller();
```
5. entry_address

`entry_address` 获得入口合约地址,比如有这样的应用场景，合约A通过合约B调用合约C的方法，此时，在合约C中就可以通过该方法拿到合约A的地址，
调用示例
```
let t = runtime::entry_address();
```
6 current_blockhash

`current_blockhash` 获得当前区块的hash,示例
```
let t = runtime::current_blockhash();
```
7. current_txhash

`current_txhash`获得当前区块的hash,示例
```
let t = runtime::current_txhash();
```
8. sha256

`sha256`计算输入参数的hash256值
```
let h = runtime::sha256("test");
```
9. check_witness

`check_witness(from)`校验是否含有该地址的签名
* 验证当前的函数调用者是不是含有from的签名 。若是（即签名验证通过），则函数返回true。
* 检查当前函数调用者是不是一个合约。若是合约，且是从该合约发起去执行函数，则返回true。即，验证 from 是不是`caller`的返回值。其中，`caller()`函数可以得到调用当前智能合约的合约哈希值。
```
assert!(runtime::check_witness(from));
```

10. notify

`notify`函数将合约中事件推送到全网，并将其内容保存到链上,调用方法如下：
```
runtime::notify("notify".as_bytes())
```
在合约中推送事件的时候，可以自定义一个事件函数，加上`#[event]`注解即可，我们的工具库中提供了该属性宏，需要通过`use ostd::macros::event;`引入。
示例如下:
```
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

11. panic

`panic`方法,可以在合约执行发生致命错误的时候立即终止交易的执行，然后回滚当前的交易。
该方法在跨合约调用的场景很重要，比如如下的应用场景，合约A中的方法a调用合约B中的方法b的场景,其中合约A的a方法在调用合约B的b方法之前，会保存一些数据到链上，但是在调用合约B的b方法时，发生了了致命的错误，需要回滚合约A中a方法执行过程中保存的数据，此时就需要在合约B的b方法中应用`panic`方法实现该功能。
```
runtime::panic("test");
```
