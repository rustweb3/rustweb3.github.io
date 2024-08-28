# 生成助记词

聊到 web3，就不得不说一个重要概念： **助记词**。
通过助记词，通过秘钥派生算法，可以完成钱包子账户的派生。
完全可以这么说: 控制一个助记词，等于控制了钱包中所有的账户。

助记词一般为一组单词，词库由 BIP39 提供。 单词的长度可以为 12 个，15 个，18 个，21 个，24 个。

## 安装

rust 中，通过 `bip39` 的 crate 可以完成助记词的生成和使用。根据不同的语言，需要开启不同的 features。

比如：

- 中文简体：`chinese-simplified`
- 繁体中文：`chinese-traditional`

同时为了，可以随机生成助记词，需要开启 `rand_core` 的 feature。

```shell
cargo add bip39 --features "chinese-simplified rand_core"
cargo add rand
```

## 生成助记词

助记词生成，需要一个随机数发生器。指定语言，指定助记词长度即可。

```rust
use bip39::{Language, Mnemonic};

fn main() {
    let mut rng = rand::thread_rng();
    let words = Mnemonic::generate_in_with(&mut rng, Language::English, 12).unwrap();
    println!("{}", words);
}
```

## 助记词导入及使用

助记词导入，直接使用 `from_str` 方法即可。使用的时候，可以指定一个 password 来增加安全性。

```rust
let words = Mnemonic::from_str(
    &"rural soup rose assist derive isolate lobster receive seek guilt verify glow",
)
.unwrap();
println!("{:?}", words);

let seeds = words.to_seed("abc");
println!("{:?}", hex::encode(seeds));
```

大多数的 web 钱包都会使用助记词的方式来管理秘钥。
但是，考虑到导入助记词过程的复杂性，很多钱包都直接使用了 空的 password 来导入助记词。
