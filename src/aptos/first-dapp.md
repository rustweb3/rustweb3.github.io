# First Dapp

我们将完成这样一个简单的dapp,并完成合约调用。
dapp 的逻辑关系是这样的。

1. 用户(address）可以通过合约，mint 一个 Counter 对象。Counter 内部包含了一个数字 value ，一个字符串消息msg，同时锁定了0.1标准的代币。
2. 用户(address) 可以调用合约修改Counter对象内部的value 的值。
3. 用户(address) 可以销毁这个Counter对象，并取回其中锁定的代币。

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