# bip32 & bip44

通过上一节 bip39 中定义的助记词，我们可以产生一个种子。种子可以派生秘钥，秘钥可以派生地址。
这样，web3 的分层钱包系统也就建立起来了。

## bip32

最开始提出的是 bip32 的标准，通过递归的方式，可以产生一个秘钥树。
但是， BIP32 并没有定义明确的路径结构。这意味着，不同币种之间可能会发生冲突。
同时，如果在一个钱包中支持多币种，BIP32 没有提供一个可靠的方案。

## bip44

所以，在 bip32 的基础上，提出了 bip44 的标准。同时，也给出了各个模块的定义标准:

一个实例 : m/44'/60'/0'/0/0

各个部分定义如下:

- m/44' 表示使用 bip44 标准
- 60' 表示使用以太坊
- 0' 表示使用主网
- 0 表示使用第一个账户
- 0 表示使用第一个地址

## 程序实例

通过 bip39 的助记词，借助 bip44 标准，派生 evm 地址

安装基础类库：

```shell
cargo add bip32
cargo add sha3
```

以下是一个，通过 bip44 派生 evm 地址的实例。

```rust
use {
    bip32::{DerivationPath, PublicKey, XPrv},
    bip39::Mnemonic,
    hex,
    sha3::Digest,
    std::str::FromStr,
};

fn main() {
    // let mut rng = rand::thread_rng();
    // let words = Mnemonic::generate_in_with(&mut rng, Language::English, 12).unwrap();
    let words = Mnemonic::from_str(
        "giant fever unveil bench mass tourist green spoon song scissors goat thumb",
    )
    .unwrap();
    println!("current words: {}", words.to_string());

    // use the words to generate a evm address
    let seeds = words.to_seed("");
    // let root_xprv = XPrv::new(&seeds).unwrap();

    let evm_derive_path = DerivationPath::from_str("m/44'/60'/0'/0/0").unwrap();
    // Xprv is ExtendedPrivateKey<k256::ecdsa::SigningKey>, so you can get it by calling private_key()
    let evm_xprv = XPrv::derive_from_path(seeds, &evm_derive_path).unwrap();
    let evm_public_key = evm_xprv.private_key().verifying_key();

    println!("evm public key: {}", hex::encode(evm_public_key.to_bytes()));

    let mut kh = sha3::Keccak256::new();
    kh.update(&evm_public_key.to_encoded_point(false).as_bytes()[1..]);
    let hash = kh.finalize().to_vec();
    let evm_address = &hash[12..];
    println!("evm address: 0x{}", hex::encode(evm_address));
}

```

部分代码说明:

- DerivationPath::from_str 可以定义推导路径，这部分，不同的币种有不同的推导路径。
- 通过 XPrv::derive_from_path 可以派生出私钥，这部分是一组扩展秘钥，可以转化为各种需要的秘钥格式。实例中 Xprv 是 ExtendedPrivateKey<k256::ecdsa::SigningKey> 类型，所以，可以调用 private_key() 获取私钥。
- 通过 private_key() 可以获取私钥，通过 verifying_key() 可以获取公钥。
- 通过公钥，进行 keccak256 哈希运算，获取 evm 地址。
