# dotenv

我们在开发过程中，需要使用到一些敏感信息，比如私钥、助记词、API 密钥等。

这些信息如果直接写在代码中，可能会导致信息泄露，造成不必要的损失。
一般情况下，我们会将这些信息写在 `.env` 文件中，然后在代码中加载这些信息。

同时，这个 .env, 要被我们加在 .gitignore 、.dockerignore 等这些同步的配置文件中，避免将敏感信息提交或同步到其他的仓库中。

## 安装

```bash
cargo add dotenv
```

## 代码初始化加载

默认从 .env 加载，也可以使用 from_path 指定路径加载。

```rust
use dotenv::dotenv;
use std::path::Path;

fn main() {
    // 默认加载 .env 文件
    dotenv().ok();

    // 从指定路径加载
    const path = Path::new("./path/to/.env");
    dotenv::from_path(path).ok();

}
```

## 获取变量的值

获取值，默认的得到的是 string 类型。

```rust
let api_key = std::env::var("API_KEY").expect("API_KEY must be set");
```

不过，可以使用 unwrap 的串联，转化数据类型。

```rust
let ok: bool = env::var("ok").unwrap().parse().unwrap();
```

## 使用实例

```rust
use dotenv::dotenv;

fn main() {
    dotenv().ok();

    // 必须包含的变量
    let api_key = std::env::var("API_KEY").expect("API_KEY must be set");
    println!("API_KEY: {}", api_key);

    // 指定默认值
    let port = std::env::var("PORT").unwrap_or_else(|_| "3000".to_string());
    println!("PORT: {}", port);

    let ok: bool = env::var("ok").unwrap().parse().unwrap();
    println!("ok -> : {}", ok);

}
```
