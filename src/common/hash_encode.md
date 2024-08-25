# hash & encode

## encode

### hex

先简单介绍编码，web3 世界中大部分数据操作 (hash,signature) 都是基于字节的。而处理字节数组很不直观。大部分情况下，使用 hex 编码来处理字节数组。

安装 hex 编码库

```bash
cargo add hex
```

一个基本的 encode 、decode 示例

```rust
fn main() {
    let msg = b"Hello, world!";
    let result = hex::encode(msg);
    println!("{}", &result);

    let msg_x = hex::decode(result).unwrap();
    println!("{}", String::from_utf8(msg_x).unwrap());
}
```

将字节数组还原成 string 的时候，需要使用 `String::from_utf8(msg_x).unwrap()` 来还原。
同时，因为结果的不确定性，所以，会返回一个 Result 类型。

## hash

hash 是 web3 中常用的操作，用于将数据转换为固定长度的字节数组。
一般用来唯一识别数据的完整性。

简单的 sha hash , 包含: sha256, sha384, sha512 等。
使用通用的类库 sha2 来实现。

```shell
cargo add sha2
```

其中 sha2 的 crates 中包含 sha256, sha384, sha512 等。
因，Hash 计算，大多数会返回不可读的字节数组，所以，一般都会和 hex 编码一起使用。

下边是一个计算 sha256 的示例

```rust
use sha2::Digest;

fn main() {
    let msg = "Hello, world!";
    let mut h = sha2::Sha256::new();
    h.reset();
    h.update(msg);
    let hash = h.finalize();
    println!("{}", hex::encode(hash));
}

```

计算 Hash 之前，需要生成对应的编码器 ： let mut h = sha2::Sha256::new()， 同时编码器，必须包含 mut 属性。
然后，将需要计算的数据写入编码器中，最后，调用 finalize 方法，获取计算结果。

**如果 Hash 计算器，需要多次计算。那么，需要先调用 reset 方法，重置编码器。**

### bcs

bcs 编码是一个二进制编码，可以把一个结构体编码成字节数组。方便数据传递。

安装 bcs 编码库

```bash
cargo add bcs
```
