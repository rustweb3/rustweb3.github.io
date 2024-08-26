# hash & encode

## encode

### hex

先简单介绍编码，编码可以把字节流(所有数据都可以是字节形式)显示成可视字符。
web3 世界中大部分数据操作 (hash,signature) 也是以字节为基础的。
这就使得，数据的可读性差一些。引入编码，就可以解决这个问题, 其中以 hex 编码居多。

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

### bcs

bcs 编码是一个二进制编码，可以把一个 满足要求的 结构体编码成字节数组，同时，也可以将一个字节流反序列化成一个结构体。
方便数据安全高效的传递，web3 中会用到 bcs 传递参数和返回结果。

安装 bcs 编码库

```bash
cargo add bcs
cargo add serde
```

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
struct Payload {
    value: u8,
    enabled: bool,
    msg: String,
}

fn main() {
    let p = Payload {
        value: 123,
        enabled: false,
        msg: "hello world".to_string(),
    };

    let d = bcs::to_bytes(&p).unwrap();
    println!("bcs encode {}", hex::encode(&d));
    let x = bcs::from_bytes::<Payload>(&d).unwrap();
    println!("value is : {}", x.value);
}


```

需要序列化的结构体，必须满足 Serialize，需要反序列化的结构体，必须满足 Deserialize 的 trait 。
可以通过 `serde::{Deserialize, Serialize};` 的宏来快速实现。

## hash

hash 是 web3 中常用的操作，用于将数据转换为固定长度的字节数组。
一般用来唯一识别数据的完整性。

### 简单的 sha hash , 包含: sha256, sha384, sha512 等。

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

### hmac

在 Sha hash 的基础上，附加一个 key ,除了完成自身的数据完整性以外，还可以添加一层简单的身份认证。
这种算法，在身份认证方面，也应用较为广泛，数据发送者，与数据传递着、数据接受者之间，可以用一个 key 来做验证。

安装

```shell
cargo add hmac
```

计算 hmac hash 的实例:

```rust
let mut hx = hmac::Hmac::<sha2::Sha256>::new_from_slice(b"hello").unwrap();
hx.update(b"world");
let b = hx.finalize().into_bytes();
println!("{}", hex::encode(b));
```

指定一个 key , 然后写入数据，通过 finalize 获取对应的字节流。

hmac 在 后续的 slip10 中被用来生成子账号的秘钥。
