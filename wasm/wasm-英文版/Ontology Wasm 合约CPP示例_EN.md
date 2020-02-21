# Ontology WASM Smart Contract Illustration

## 1. Writing Smart Contracts in C++

Much similar to EOS. developers can use C++ to develop smart contracts on the Ontology platform. Let us take a look at a sample **Hello World** app.

Sample Code:

```cpp
#include<ontiolib/ontio.hpp>
#include<stdio.h>

using namespace ontio;
class hello:public contract {
    public:
    using contract::contract:
    void sayHello(){
        printf("hello world!");
    }
};
ONTIO_DISPATCH(hello, (sayHello));
```
### Smart Contract Entry Point

The [Ontology WASM CDT Compiler](https://github.com/ontio/ontology-wasm-cdt-cpp) encapsulates the required features for entry point and parameter processing. Thus, developers do not need to define entry point methods.

The entry point method can be called in the following manner:

```cpp
ONTIO_DISPATCH(hello, (sayHello));
```

The next important part of the contract would be the external interface. This interface would make the contract's services available to external parties. In the sample code above we use the `sayHello()` method to demonstrate the same.

```cpp
 printf("hello world!");
```

This "Hello World" will be printed out in the node log records at the "debug level". Practically speaking, the `printf()` method can only be used for debugging. A more realistic smart contract would need implement many more complex features.

### Smart Contract API

Ontology Wasm 提供如下API与区块链的底层进行交互

Ontology WASM provides an API that contains the following methods that allow communication with the blockchain system.

|        API        |                    Parameter                     | Return Value | Description                                                                   |
| :---------------: | :----------------------------------------------: | :----------: | ----------------------------------------------------------------------------- |
|     timestamp     |                       None                       |    uint64    | Current UNIX timestamp                                                        |
|   block_height    |                       None                       |    uint32    | Current block height                                                          |
|   self_address    |                      Nones                       |   address    | Contract address                                                              |
|  caller_address   |                       None                       |   address    | Invocation address (Same as `self_address` if invocation is not external)     |
|   entry_address   |                       None                       |   address    | Entry contract address (Same as `self_address` if invocation is not external) |
|   check_witness   |                     address                      |     bool     | Check the signature of the incoming address                                   |
| current_blockhash |                       None                       |     H256     | Current block hash                                                            |
|  current_txhash   |                       None                       |     H256     | Current transaction hash                                                      |
|      notify       |                      string                      |     void     | Send even notification                                                        |
|    call_native    |             address, params, result              |     void     | Invoke native contract                                                        |
|   call_contract   |             address, params, result              |     void     | Invoke ordinary contract (WASM/NeoVM)                                         |
|    storage_get    |                   key, result                    |     void     | Fetch stored data                                                             |
|    storage_put    |                    key, value                    |     void     | Write data on the chain                                                       |
|  storage_delete   |                       key                        |     void     | Delete stored data                                                            |
|  contract_create  | code, vmtype, name, version, author, email, desc |   address    | Create new contract                                                           |
| contract_migrate  | code, vmtype, name, version, author, email, desc |   address    | Migrate (upgrade) contract                                                    |
|  contract_delete  |                     address                      |     void     | Delete contract                                                               |

Let us develop a slightly more complicated WASM contract to demonstrate how to use the API.

### Red Envelope Smart Contract

Giving and receiving red envelopes is a part ogf China's tradition on important festivals and occassions. Red envelopes can now be sent and received using many different tools including social networking and IM platforms such as WeChat. The amount collected can also be deposited to bank accounts.

Let us try an create a smart contract that works the way WeChat's red envelope mechanism works. **ONT**, **ONG**, and other OEP-4 standard cryptocurrencies and tokenized assets can be transferred in the form of red envelopes. The amount received is transferred to the receiver's wallet account.

#### 1. Creating a New Contract

```cpp
#include<ontiolib/ontio.hpp>

using namespace ontio;

class redEnvlope: public contract{
    
}
ONTIO_DISPATCH(redEnvlope, (createRedEnvlope)(queryEnvlope)(claimEnvlope));
```

First, we need to create a new contract file and rename it to `redEnvelope.cpp`.

In this contract we will be implementing three API methods.

`createRedEnvlope`  : Create red envelope

`queryEnvlope` : Query red envelope details

`claimEnvlope` : Claim red envelope

```cpp
    std::string rePrefix = "RE_PREFIX_";
    std::string sentPrefix = "SENT_COUNT_";
    std::string claimPrefix = "CLAIM_PREFIX_";
```

We need to store certain imporant data. Data is store in the form of key-value pairs withing the scope of the contract. The **key** to these data need to set the prefix to make querying more convenient.


```cpp
    address ONTAddress = {0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1};
    address ONGAddress = {0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,2};
```

Since our contract supports the two native assets **ONT** and **ONG**, we can predefine the contract addresses of these these two assets.

> Note: Unlike a standard smart contact where the contract address is calculated based on contract hash, the address of the native contract is fixed.

```cpp
    struct receiveRecord{
        address account;   // user address
        asset amount;      // claimed amount  
        ONTLIB_SERIALIZE(receiveRecord,(account)(amount))
    };

    struct envlopeStruct{
        address tokenAddress;   // asset token address
        asset totalAmount;      // red envelope total amount
        asset totalPackageCount; // total no. of red envelopes
        asset remainAmount;      // remaining amount
        asset remainPackageCount; // remaining 
        std::vector<struct receiveRecord> records;  // claim records
        ONTLIB_SERIALIZE( envlopeStruct,  (tokenAddress)(totalAmount)(totalPackageCount)(remainAmount)(remainPackageCount)(records) )
    };
```

We need to store the red envelope information in the contract, such as asset related information (token contract address, red envelope total amount, no. of red envelopes, etc.)

```
ONTLIB_SERIALIZE(receiveRecord,(account)(amount))
```

The macros defined in **CDT** can be used to serialize the `struct` type data before storage.

The preparation is almost complete, and at this point we can start coding the specific API logic.

#### 2. Creating a Red Envelope

```cpp
bool createRedEnvlope(address owner,asset packcount, asset amount,address tokenAddr ){

        return true;
    }
```

1. A new red envelope is created by defining the creator's address, no. of red envelopes, red envelope amount, and the asset token address. 

```cpp
ontio_assert(check_witness(owner),"checkwitness failed");
```

2. Verify whether the creator has signed the action, and rollback the transaction if not.

> Note: `ontio_assert(expr, errormsg)` Here, when `expr` is `false`, an exception is thrown and the process is quit.

```cpp
		if (isONTToken(tokenAddr)){
            ontio_assert(amount >= packcount,"ONT amount should greater than packcount");
        }
```

3. If the red envelope asset is ONT, the amount of ONT must be equal to or greater than the no. of red envelopes, since the asset is indivisible (cannot be smaller than 1). So this condition ensures thaat each red envelope contains at least 1 ONT.

```cpp
        key sentkey = make_key(sentPrefix,owner.tohexstring());
        asset sentcount = 0;
        storage_get(sentkey,sentcount);
        sentcount += 1;
        storage_put(sentkey,sentcount);
```

4. For every red envelope creator, we need to maintain the record of the total number of red envelopes created and sent by them.

```cpp
        H256 hash ;

        hash256(make_key(owner,sentcount),hash) ;

        key rekey = make_key(rePrefix,hash256ToHexstring(hash));
```

5.  A red envelope hash is generated for each red envelope. This hash value serves as a unique identifier.


```cpp
	 	address selfaddr = self_address();
		if (isONTToken(tokenAddr)){

            bool result = ont::transfer(owner,selfaddr ,amount);
            ontio_assert(result,"transfer native token failed!");
           
        }else if (isONGToken(tokenAddr)){

            bool result = ong::transfer(owner,selfaddr ,amount);
            ontio_assert(result,"transfer native token failed!");
        }else{
            std::vector<char> params = pack(std::string("transfer"),owner,selfaddr,amount);
            bool res; 
            call_contract(tokenAddr,params, res );

            ontio_assert(res,"transfer oep4 token failed!");
        }
```

6. A tokenized asset it imported to the contract based on it's type. The `self_address()` method can be used to fetch the address of the invocation. The amount of token imported to the contract is based on the token type input by the user.


> Note: The transfer operation for the two native assets ONT and ONG is carried out using the `ont::transfer` API method provided in the CDT. Other **OEP-4** based tokens need to be transferred using the standard cross-contract invocation methods.

> Note: A smart contract address can receive assets of any type, just as a wallet address can. However, the contract address is generated based on the binary code hash of the contract code, and so the assets stored in the contract address cannot be manipulated without the corresponding private key. Hence, If asset related operations are not set within the contract, there is no way to control the assets stored in it.

```cpp
		struct envlopeStruct es ;
        es.tokenAddress = tokenAddr;
        es.totalAmount = amount;
        es.totalPackageCount = packcount;
        es.remainAmount = amount;
        es.remainPackageCount = packcount;
        es.records = {};
        storage_put(rekey, es);
```

7. The contract information is saved in the storage.

```cpp
        char buffer [100];
        sprintf(buffer, "{\"states\":[\"%s\", \"%s\", \"%s\"]}","createEnvlope",owner.tohexstring().c_str(),hash256ToHexstring(hash).c_str());

        notify(buffer);
        return true;
```

8. The red envelope creation event is sent to the chain, since this is an asyncrhonous event with respect to contract invocation. Upon successful execution, the contract notifies the client regarding the event. The specific format of this notification can be defined by the developer.

With this a simple red envelope is created. The next step is to implement the method that can query the red envelope information.

#### 2. Querying Red Envelope Information

```cpp
   std::string queryEnvlope(std::string hash){
        key rekey = make_key(rePrefix,hash);
        struct envlopeStruct es;
        storage_get(rekey,es);
        return formatEnvlope(es);
    }
```


The query logic for the red envelope is simple. The stored red envelope data needs to be fetched, formatted and returned.


And finally, the users can claim the red envelope based on the red envelope hash (an ID).

> Note: For read-only operations of the smart contract, such as information query, the results canbe fetched using pre-execution. Unlike the normal execution process, pre-execution does not require wallet authentication and does not consume **ONG**.

#### 3. Claiming Red Packet

We have succesfully imported the assets to the smart contract. At this point, the ID can be shared with other users and they can start claiming the red envelope.

```cpp
  bool claimEnvlope(address account, std::string hash){
  	return true;
  }
```

1. Claiming a red envelope requires the claiming party's account address and the red envelope's **hash**.

```cpp
ontio_assert(check_witness(account),"checkwitness failed");
key claimkey = make_key(claimPrefix,hash,account);
asset claimed = 0 ;
storage_get(claimkey,claimed);
ontio_assert(claimed == 0,"you have claimed this envlope!");
```

2. Similarly, the claiming party's signature needs to be verified to ensure that a person may only claim a red envelope themselves, and not by proxy. Also, each user is allowed to claim a red envelope only once.

```cpp
        key rekey = make_key(rePrefix,hash);
        struct envlopeStruct es;
        storage_get(rekey,es);
        ontio_assert(es.remainAmount > 0, "the envlope has been claimed over!");
        ontio_assert(es.remainPackageCount > 0, "the envlope has been claimed over!");
```

3. Using the red envelope hash it's data can be fetched and can be determine whether a red envelope has been fully claimed. 

```cpp
        struct receiveRecord record ;
        record.account = account;
        asset claimAmount = 0;
```

4. The claim is added to the claim records.

```cpp
		if (es.remainPackageCount == 1){
            claimAmount = es.remainAmount;
            record.amount = claimAmount;
        }else{
            H256 random = current_blockhash() ;
            char part[8];
            memcpy(part,&random,8);
            uint64_t random_num = *(uint64_t*)part;
            uint32_t percent = random_num % 100 + 1;

            claimAmount = es.remainAmount * percent / 100;
            //ont case
            if (claimAmount == 0){
                claimAmount = 1;
            }else if(isONTToken(es.tokenAddress)){
                if ( (es.remainAmount - claimAmount) < (es.remainPackageCount - 1)){
                    claimAmount = es.remainAmount - es.remainPackageCount + 1;
                }
            }

            record.amount = claimAmount;
        }
        es.remainAmount -= claimAmount;
        es.remainPackageCount -= 1;
        es.records.push_back(record);
```

5. This part of the program logic is a little lengthy. Here, the logic is to calculate the amount of asset claimed from the red envelope. If it is the last red packet, then the amount left is the amount in the last red packet. Otherwise, the remaining asset amount is calculated using a random number generated using the current block hash. This remaining asset amount is then updated in the red envelope information.



```cpp
        address selfaddr = self_address();
        if (isONTToken(es.tokenAddress)){
            bool result = ont::transfer(selfaddr,account ,claimAmount);
            ontio_assert(result,"transfer ont token failed!");
        } else  if (isONGToken(es.tokenAddress)){
            bool result = ong::transfer(selfaddr,account ,claimAmount);
            ontio_assert(result,"transfer ong token failed!");
        } else{
            std::vector<char> params = pack(std::string("transfer"),selfaddr,account,claimAmount);

            bool res = false; 
            call_contract(es.tokenAddress,params, res );
            ontio_assert(res,"transfer oep4 token failed!");
        }
```

6. Based on the claimed asset, the calculated amount of the corresponding asset is transferred to the claim account address from the contract.



```cpp
        storage_put(claimkey,claimAmount);
        storage_put(rekey,es);
        char buffer [100];        
        std::sprintf(buffer, "{\"states\":[\"%s\",\"%s\",\"%s\",\"%lld\"]}","claimEnvlope",hash.c_str(),account.tohexstring().c_str(),claimAmount);

        notify(buffer);
        return true;
```

7. The claim records are stored and the updated red envelope information is stored along with the notification of the event being sent out.
As stated above, the `claimEnvlope()` method is the only way to move an asset out of this contract. Hence, we establish that the assets stored in the contract are safe.
The simple logic for the red envelope system is complete. The entire smart contract sample code is available [here](https://github.com/JasonZhouPW/pubdocs/blob/master/redEnvlope.cpp) for reference.


### Testing the Smart Contract

1. Using the CLI

Please refer to： <https://github.com/ontio/ontology-wasm-cdt-cpp/blob/master/How_To_Run_ontologywasm_node.md>


2. Using the Golang SDK

Please refer to: <https://github.com/ontio/ontology-wasm-cdt-cpp/blob/master/example/other/main.go>


This example serves to demonstrate how a complete Ontology WASM contract uses API methods to interact with the blockchain. If a complete product is to be developed using this technology, there would be several other considerations such as privacy related concerns for the red envelope. Anyone can monitor a red envelope events to get the hash, and then claim it. This issue can be solved by fixing the addresses, and thereby the users who can claim a red packet. Interested developers can make the necessary changes to the code and test it out.



