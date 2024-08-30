# bip32 & bip44 & slip10

通过上一节 bip39 中定义的助记词，我们可以产生一个种子。种子可以派生秘钥，秘钥可以派生地址。
这样，web3 的分层钱包系统也就建立起来了。

## bip32

最开始提出的是 bip32 的标准，通过递归的方式，可以产生一个秘钥树。
但是， BIP32 并没有定义明确的路径结构，路径只要符合一个简单的字符串即可。 比如: m/0/1/2
各模块之间可以随意定义，所以，如果在一个钱包中支持多币种，BIP32 没有提供一个可靠的方案。
为了解决这个问题，提出了 bip44 的标准，给各个模块定义了标准。

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

## SLIP10

很多公链（比如 SUI、Aptos、Solana）采用的是 ed25519 算法，原始的 bip32 并没有定义 ed25519 的秘钥派生方式。
SLIP10 是 BIP32 的改进版，定义了 ed25519 的秘钥派生方式。生成代码逻辑如下:

```rust
use bip32::DerivationPath;
use hmac::{Hmac, Mac};
use sha2::Sha512;

pub fn derive_ed25519_private_key_by_path(seed: &[u8], path: DerivationPath) -> [u8; 32] {
    let indexes = path
        .into_iter()
        .map(|i: bip32::ChildNumber| i.into())
        .collect::<Vec<_>>();
    derive_ed25519_private_key(seed, &indexes)
}

#[allow(non_snake_case)]
fn derive_ed25519_private_key(seed: &[u8], indexes: &[u32]) -> [u8; 32] {
    let mut I = hmac_sha512(b"ed25519 seed", &seed);
    let mut data = [0u8; 37];

    for i in indexes {
        let hardened_index = 0x80000000 | *i;
        let Il = &I[0..32];
        let Ir = &I[32..64];

        data[1..33].copy_from_slice(Il);
        data[33..37].copy_from_slice(&hardened_index.to_be_bytes());

        //I = HMAC-SHA512(Key = Ir, Data = 0x00 || Il || ser32(i'))
        I = hmac_sha512(&Ir, &data);
    }

    I[0..32].try_into().unwrap()
}

pub fn hmac_sha512(key: &[u8], data: &[u8]) -> [u8; 64] {
    type HmacSha512 = Hmac<Sha512>;
    let mut hmac = HmacSha512::new_from_slice(key).expect("HMAC can take key of any size");
    hmac.update(data);
    let result = hmac.finalize();
    result.into_bytes().try_into().unwrap()
}

```
