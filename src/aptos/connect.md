
# 连接到 Aptos 网络

Aptos 网络官方提供的 RPC 地址被封装在 `aptos_sdk::rest_client::AptosBaseUrl` 中。

网络设置为官方连接，分为以下几种。

| Name    | Fullnode URL                             | Faucet URL                             |
| ------- | ---------------------------------------- | -------------------------------------- |
| Mainnet | `https://fullnode.mainnet.aptoslabs.com` | N/A                                    |
| Devnet  | `https://fullnode.devnet.aptoslabs.com`  | `https://faucet.devnet.aptoslabs.com`  |
| Testnet | `https://fullnode.testnet.aptoslabs.com` | `https://faucet.testnet.aptoslabs.com` |
| Custom  | `Custom URL provided by the user`        | N/A                                    |

通过 `aptos_sdk::rest_client::Client` 提供连接对象，提供 Url 参数。

```rust
let client = Client::new(AptosBaseUrl::Testnet.to_url());
```

client 提供 rpc 信息，其中有几个信息，你会很快用到

1. 链的 ID : `chain_id`

```rust
let index_info = client.get_index().await?;
info!("chain_id info: {:?}", index_info.inner().chain_id);
```

2. 账户 `sequence_number`

```rust
let sequence = client.get_account(import_account.address()).await?;
info!("account sequence is : {}",sequence.inner().sequence_number);
```

这一点 `aptos` 和其他的公链有所区别。如果是新的账户，那么 sequence_number 则无法获取。同时这个账户也无法进行任何的操作。
解锁办法，往其中发送任意的 apt 激活即可。

## 产生随机账户

一般随机产生秘钥，然生产生随机账号用于测试。
导入 rand 和 rand_core 的时候，建议和 aptos 官方使用的一致。可以在 sdk 的 [Cargo.toml](https://github.com/aptos-labs/aptos-core/blob/main/Cargo.toml) 中查看 。
目前的版本是:

```toml
rand = "0.7.3"
rand_core = "0.5.1"
```

账号操作集中在 `aptos_sdk::types::LocalAccount` 中，这里可以产生一个 signer 对象，可以和 以上的 client 直接发起交易。

产生随机账户,并打印地址,及 私钥：

```rust
let mut alice_account = LocalAccount::generate(&mut OsRng);
info!(
    "Alice account private key is : 0x{}",
    hex::encode(alice_account.private_key().to_bytes())
);
info!("Alice account address is : {}", alice_account.address());
```

其中产生的私钥的 hex 编码，可以用于导入其他的 aptos 钱包，同时也可以通过 LocalAccount 导入，供以后使用。

## 导入账户

通过 `LocalAccount::from_private_key` 的操作，可以导入账户。但是，导入账户的时候，无法知道用户的 sequence_number。
所以，需要从链上获取更新一次，供以后使用。

```rust
let client = Client::new(AptosBaseUrl::Testnet.to_url());
let mut import_account = LocalAccount::from_private_key(
    "0xb35ea8e82ec4daebab0892a466396bc5276e9d2fd05bbf9d4c5e3cacb0b90f68",
    0,
)
.unwrap();
let account_info = client.get_account(import_account.address()).await?;
info!(
    "Import account sequence is : {}",
    account_info.inner().sequence_number
);
import_account.set_sequence_number(account_info.inner().sequence_number);
```

## 获取测试代币

devnet 和  testnet 都包含了水龙头的地址，可以获取测试代币。
使用水龙头的 Url 和 rest_client 构建一个 FaucetClient ，调用 fund 方法用来申请测试代币。

```rust
let faucet_url = url::Url::parse("https://faucet.testnet.aptoslabs.com")?;
let faucet_client = FaucetClient::new_from_rest_client(faucet_url, client.clone());
let faucet_result = faucet_client
    .fund(import_account.address(), 100_000_000 * 5)
    .await?;
info!("Faucet result: {:?}", faucet_result);
```

## 获取账户余额

有几种方法获得账户的余额。

1. 使用 client 直接获取

```rust
let balance = client.get_account_balance(account.address()).await?;
info!("current balance is : {:?}", balance.inner().coin);
```

2. 使用 coin_client 获取

```rust
let coin_client = CoinClient::new(&client);
let coin_balance = coin_client.get_account_balance(&account.address()).await?;
info!("coin balance is : {:?}", coin_balance);
```

3. 更加通用的方式是使用 client.get_account_resource 方法获取。以上的两种方法其实也是使用这种方法获取的。 

```rust
let another_resource = client
    .get_account_resource(
        account.address(),
        "0x1::coin::CoinStore<0xa8a5e68261a0f198e34deb2c0fd2683244e51dd015b8cb6987efefc61708d76a::Kana::Kana>",
    )
    .await?;

match another_resource.inner() {
    Some(resource) => {
        let coin = resource.data.get("coin").unwrap();
        info!("another resource is : {:?}", coin.get("value"));
    }
    None => {
        info!("another resource is not found");
    }
}
```

这种方法也适用于获取账户的资源对象解析。

## 发起转账
