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

注意: 因为 aptos sdk 依赖了整个的 aptos crates , 所以，第一次安装很慢，情耐心等待。

## 配置 build 选项

编辑 .cargo/config.toml 如果没有，创建一个，添加 build 选项

写入如下内容:

```toml
[build]
rustflags = ["--cfg", "tokio_unstable"]
```
