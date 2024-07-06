
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

## 查询余额