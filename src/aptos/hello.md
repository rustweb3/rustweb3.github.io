# Aptos 开发基础配置

在开始 `aptos` 开始之前需要安装 sdk。

aptos 的 sdk 安装相对特殊一些，除了安装 dependencies，还需要打对应的 patch 同时配置独立的编译参数。

## 安装配置 cargo 依赖

可以使用 cargo 命令安装:

```shell
cargo add aptos-sdk --git https://github.com/aptos-labs/aptos-core --branch devnet
```

或者 也可以直接修改 Cargo.toml 文件:

```toml
[dependencies]
aptos-sdk = { git = "https://github.com/aptos-labs/aptos-core", branch = "devnet" }
```

添加 patch  

```toml
[patch.crates-io]
merlin = { git = "https://github.com/aptos-labs/merlin" }
x25519-dalek = { git = "https://github.com/aptos-labs/x25519-dalek", branch = "zeroize_v1" }
```

注意: 因为aptos sdk 依赖了整个的 aptos crates , 所以，第一次安装很慢，情耐心等待。

## 配置build 选项

编辑 .cargo/config.toml 如果没有，创建一个，添加 build 选项

写入如下内容:

```toml
[build]
rustflags = ["--cfg", "tokio_unstable"]
```

## 连接到 Aptos 网络

Aptos 网络官方提供的RPC 地址被封装在 `aptos_sdk::rest_client::AptosBaseUrl` 中。

网络设置为官方连接，分为以下几种。

| Name    | Fullnode URL                                | Faucet URL                                |
|---------|---------------------------------------------|-------------------------------------------|
| Mainnet | `https://fullnode.mainnet.aptoslabs.com`    | N/A                                       |
| Devnet  | `https://fullnode.devnet.aptoslabs.com`     | `https://faucet.devnet.aptoslabs.com`     |
| Testnet | `https://fullnode.testnet.aptoslabs.com`    | `https://faucet.testnet.aptoslabs.com`    |
| Custom  | `Custom URL provided by the user`           | N/A                                       |

通过 `aptos_sdk::rest_client::Client` 提供连接对象，提供 Url 参数。

```rust
let client = Client::new(AptosBaseUrl::Testnet.to_url());
```

client 提供 rpc 信息，其中有几个信息，你会很快用到 

1. 链的ID : `chain_id`

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