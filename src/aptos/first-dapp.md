# First Dapp

我们将完成这样一个简单的dapp,并完成合约调用。
dapp 的逻辑关系是这样的。

1. 用户(address）可以通过合约，mint 一个 Counter 对象。
2. Counter 内部包含了一个数字 value ，一个字符串消息msg，同时锁定了0.1标准的代币。
3. 用户(address) 可以调用合约修改Counter对象内部的value 的值。
4. 用户(address) 可以销毁这个Counter对象，并取回其中锁定的代币。

交互过程中，也会产生对应的Event，我们也会通过rust 的rpc 调用来获取。

## 合约部分

```move
module counter::counter {

    use std::signer;
    use std::coin;
    use std::string::String;
    use std::timestamp;
    use std::event::emit;

    const LOCK_COIN_VALUE:u64 = 10_000_000;

    const EVENT_TYPE_CREATE:u8 = 0;
    const EVENT_TYPE_INCREMENT:u8 = 1;
    const EVENT_TYPE_DESTROY:u8 = 2;

    #[event]
    struct CounterEvent has drop,store {
        sender: address,
        value: u64,
        timestamp: u64,
        event_type: u8,
    }

    struct MyCounter<phantom CoinType> has key,store {
        value: u64,
        msg: String,
        lock_coin: coin::Coin<CoinType>
    }


    fun new_counter_event(sender:address,value:u64,event_type:u8) : CounterEvent {
        CounterEvent { sender, value, timestamp:timestamp::now_seconds(),event_type }
    }
    
    fun new_counter<CoinType>(value:u64,msg:String,sender:&signer) : MyCounter<CoinType> {
        let lock_coin = coin::withdraw<CoinType>(sender, LOCK_COIN_VALUE);
        emit(new_counter_event(signer::address_of(sender),value,EVENT_TYPE_CREATE));
        MyCounter { value ,msg, lock_coin }
    }

    public entry fun mint<CoinType>(sender:signer,value:u64,msg: String) {
        let sender_addr = signer::address_of(&sender);
        if (!exists<MyCounter<CoinType>>(sender_addr)) {
            move_to(&sender, new_counter<CoinType>(value,msg,&sender));
        }
    }

    public entry fun increment<CoinType>(sender:signer,value:u64) acquires MyCounter {
        let sender_addr = signer::address_of(&sender);
        if (exists<MyCounter<CoinType>>(sender_addr)) {
            let x = borrow_global_mut<MyCounter<CoinType>>(sender_addr);
            x.value = x.value + value;
            emit(new_counter_event(sender_addr,x.value,EVENT_TYPE_INCREMENT));
        }
    }

    public entry fun destroy<CoinType>(sender:signer) acquires MyCounter{
        let sender_addr = signer::address_of(&sender);
        if (exists<MyCounter<CoinType>>(sender_addr)) {
            emit(new_counter_event(sender_addr,0,EVENT_TYPE_DESTROY));
            let MyCounter{value:_value,msg:_msg,lock_coin} = move_from<MyCounter<CoinType>>(sender_addr);
            coin::deposit<CoinType>(sender_addr,lock_coin);
        }
    }

}
```

以上是这个dapp的智能合约，包含 一个泛型Struct MyCounter, 一个 Event Struct, 三个 entry 函数供外部操作。

## 部署

按照以下的步骤完成测试合约部署：

### 1. 初始化

```shell
aptos init 
```

根据需要选择不同的网络 ,网络包括: `devnet, testnet, mainnet, local, custom`。不同的网络账号间是不通用的。
下一步选择输入私钥或者生成一个新的。
最后，你将获得你的部署账号。把这个账户写入 合约配置文件的 address 模块。

```toml
[addresses]
counter = "0x9ce5950565b5cb8d514b09f5ae5afdd0ed75d41bcbb73409bd066378dcd4b7f3"
```

### 2. 编译:

```shell
aptos move compile --package-dir . --skip-fetch-latest-git-deps 
```

### 3. 部署

```shell
aptos move publish --skip-fetch-latest-git-deps
```

部署完成后，接下来就可以通过 合约的地址来完成调用了。

## 合约调用

合约调用的逻辑大体上分为如下的几步:

1. 获得 `entry` 调用入口，添加函数参数，泛型参数，构建 payload。
2. 在 `payload` 的基础上，添加交易过期时间、链的Id 就可以完成一个未签名的交易内容。
3. 通过 `LocalAccount` 的 `sign_with_transaction_builder` 完成签名。
4. 最后通过 `rpc client` 广播完成签名的交易，在链上执行。

### entry 入口调用定义

一大部分的合约调用都属于合约里的`entry`函数调用，通过一个账号地址定位到包的部署位置。

```rust
let package = AccountAddress::from_str(&package_addr)?;
```

调用合约，之前需要调整好 `Account` 的 `sequence_number`。

一个 `package` 地址中会发布很多的 module,所以需要调用合约entry 所在的 module 名称。通过ModuleId 来组织。

```rust
ModuleId::new(package, Identifier::new("counter").unwrap()),
```

定义为好 package 中的模块，还需要 entry function 的名称。通过 Identifier 来定义

```rust
Identifier::new("mint").unwrap()
```

### 参数组织

调用合约的参数分为两种，一种是泛型参数，一种是函数参数。两类参数都是提供一个数组传递。

1. 泛型参数通过 TypeTag::from_str 就可以获取:

```rust
vec![TypeTag::from_str(&"0x1::aptos_coin::AptosCoin")?]
```

2. 正常的函数参数，需要通过 bcs 的序列化，转为字节序列

```rust
vec![
    bcs::to_bytes(&(1024 as u64))?,
    bcs::to_bytes(&("hello world".to_string()))?,
],
```

bcs::to_bytes 的参数，一定要制定参数的类型，这个要和链上的参数类型一致，否则合约将无法识别。

### 设置交易过期时间

这个获取当前系统的时间戳，即可。

```rust
let expire_at = time::SystemTime::now()
    .duration_since(time::UNIX_EPOCH)?
    .as_secs()
    + 30;
```

**小提示:** 尽管大多数的服务器都配置NAT时间同步。但是，有时候，由于某些原因服务器的时间会出现延迟或者提前的情况。所以，需要把这个因素提前考虑进去。

### 签名交易

构建一个 `TransactionBuilder`, 加入 `payload` 、`expire_at` 、 `chain_id` 三个参数。
把构建好的 builder 传递给 client完成签名。

```rust
let builder = TransactionBuilder::new(payload, expire_at, ChainId::new(chain_id));
let txn = account.sign_with_transaction_builder(builder);
```

### 广播并等待交易

提交交易完成后，可以获取到交易的hash ,通过这个 hash 可以确认合约的执行状态。
wait_for_signed_transaction，将等待交易完成再执行下边的逻辑。

```rust
let signature = client.submit(&txn).await?;
println!("Signature: {}", signature.inner().hash);
client.wait_for_signed_transaction(&txn).await?;
```

### 完整代码

一个完整的合约调用模块如下:

```rust
let package = AccountAddress::from_str(&package_addr)?;

let txn = {
    let payload = TransactionPayload::EntryFunction(EntryFunction::new(
        ModuleId::new(package, Identifier::new("counter").unwrap()),
        Identifier::new("mint").unwrap(),
        vec![TypeTag::from_str(&"0x1::aptos_coin::AptosCoin")?],
        vec![
            bcs::to_bytes(&(1024 as u64))?,
            bcs::to_bytes(&("hello world".to_string()))?,
        ],
    ));

    let expire_at = time::SystemTime::now()
        .duration_since(time::UNIX_EPOCH)?
        .as_secs()
        + 30;

    let builder = TransactionBuilder::new(payload, expire_at, ChainId::new(chain_id));
    let txn = account.sign_with_transaction_builder(builder);
    txn
};

let signature = client.submit(&txn).await?;
println!("Signature: {}", signature.inner().hash);
client.wait_for_signed_transaction(&txn).await?;
```
