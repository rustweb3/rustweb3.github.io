# 连接到 solana 网络

## 安装基本环境

使用 solana 开发程序之前需要一些基础的 sdk，通过 cargo 安装即可。

```shell
cargo add solana-client
cargo add solana-sdk
```

随着版本变化，可能安装的版本不一致。

## 网络连接

solana 的通用网络分四种

| Environment | URL                                      |
|-------------|------------------------------------------|
| Localhost   | `http://127.0.0.1:8899`                  |
| Devnet      | `https://api.devnet.solana.com`          |
| Testnet     | `https://api.testnet.solana.com`         |
| Mainnet-Beta| `https://api.mainnet-beta.solana.com`    |

其中 Localhost 网络可以通过 `solana-test-validator -r` 启动，启动以后，将在本地监听 8899 的 rpc 端口。
devnet 和 testnet 为两种不同的测试网络，mainnet 就是主网了，牵扯真金白银的。 所以上主网之前需要在其他的网络测试完整。

网络的操作基本上都是通过 `solana_client::rpc_client::RpcClient` 来完成的。初始化也极为简单:

```rust
let client = RpcClient::new("https://api.devnet.solana.com".to_string());
```

一般情况下，你可能需要通过 quicknode 这类第三方 rpc 获得可用性较高的 rpc 节点。

## 网络请求

初始化好了客户端就可以发起rpc 请求了。
大多数网络请求都遵循 solana 的 [rpc 文档](https://solana.com/docs/rpc)。

以下是两个简单的请求，获取 链的版本号和当前区块高度。

```rust
info!("version : {}", client.get_version().unwrap());
info!("block height  : {}", client.get_block_height().unwrap());
```

## 账号结构

账号部分操作几种在 `solana_sdk::signature::{Keypair, Signer}` 。
Keypair 其实就是 ed25519_dalek 的一组 Keypir 密钥对， Signer 是对应的 trait 接口。

生成随机账户:

```rust
let pair: Keypair = Keypair::new();
```

导出私钥,导出编码过的 bytes 数据，solana 中一般使用 base58编码。

```rust
pair.to_base58_string()
```

打印地址, solana 地址即为公钥，所以，直接打印 pair.pubkey 的 base58编码 即可。

```rust
pair.pubkey.to_string()
```

同理的，base58 编码的字节都可以导入，以获得需要操作的账户。同时 私钥的 base58 编码 也可以导入到各个 solana 钱包当中。

```rust
let a_pair = Keypair::from_base58_string("base58_key");
```

## 简单的完整示例

主要完成以下工作:

1. 产生一个随机账户,打印对应的地址、私钥等信息
2. 通过 私钥信息重新导入，获取账户
3. 检查余额，如果余额为 0 ，那么 申请 airdrop （因为环境问题，或者其他的什么问题，可能 airdrop 失败，可以拿本地网络测试)

```rust
use {
    anyhow::{Ok, Result},
    clap::Parser,
    log::{debug, error, info, warn},
    solana_client::rpc_client::RpcClient,
    solana_playground::{app, cli, jobs},
    solana_sdk::signature::{Keypair, Signer},
};

#[tokio::main]
async fn main() -> Result<()> {
    app::RunTime::init();
    let mut runtime = app::RunTime::new();
    runtime.do_init(app::InitOptions {
        config_merge_env: true,
        config_merge_cli: true,
    });
    if runtime.cli.name.is_none() && runtime.cli.command.is_none() {
        error!("Please input the command");
        return Err(anyhow::anyhow!("Please input the command"));
    }

    match runtime.cli.command {
        Some(cli::Command::Test {}) => {
            let client = RpcClient::new("https://api.devnet.solana.com".to_string());
            let pair: Keypair = Keypair::new();
            let pubkey = pair.pubkey();
            warn!("private key is : {}", pair.to_base58_string());
            info!("solana address is : {}", pubkey.to_string());
            let a_pair = Keypair::from_base58_string("3a1q7fn3QPex7MkjKFpQcwP8TYUqLyZT6CYUshGRHFPj79Sq67KNJBk9tJSgAVXMNnjfLuUMDCX9epE8DAqTEY6Q");
            let a_pubkey = a_pair.pubkey();
            info!("a_pubkey address is  : {}", a_pubkey.to_string());
            let balance = client.get_balance(&a_pubkey).unwrap();
            info!("balance is : {}", balance);

            if balance == 0 {
                let tx = client.request_airdrop(&a_pubkey, 1000000000).unwrap();
                info!("tx is : {:?}", tx);
                let balance = client.get_balance(&a_pubkey).unwrap();
                info!("balance is : {}", balance);
            }
        }
        _ => {
            error!("Please input the command");
            cli::Cli::parse_from(&["rustapp", "--help"]);
            return Err(anyhow::anyhow!("Please input the command"));
        }
    }

    Ok(())
}
```