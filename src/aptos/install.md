# Aptos 开发基础配置

使用 `rust` 做 `aptos` 的开发，相对复杂一些。

## 安装配置 cargo 依赖 ，并打对应的 patch， 在项目的 `Cargo.toml` 中添加如下的选项

```toml
[dependencies]
aptos-sdk = { git = "https://github.com/aptos-labs/aptos-core", branch = "devnet" }
 
[patch.crates-io]
merlin = { git = "https://github.com/aptos-labs/merlin" }
x25519-dalek = { git = "https://github.com/aptos-labs/x25519-dalek", branch = "zeroize_v1" }
```

然后，还需要配置单独的 build 选项，创建 .cargo/config.toml

写入如下内容: 

```toml
[build]
rustflags = ["--cfg", "tokio_unstable"]
```
