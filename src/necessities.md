# 必要条件

## 稳定版 crate 的公共依赖必须稳定 (C-STABLE)

一个 crate 不能在其所有公共依赖都稳定之前成为稳定版（>=1.0.0）。

公共依赖指的是在当前 crate 的公共 API 中使用的其他 crate 类型。

```rust
pub fn do_my_thing(arg: other_crate::TheirThing) { /* ... */ }
```

包含此函数的 crate 不能稳定，除非 `other_crate` 也是稳定的。

请注意，公共依赖可能会在意想不到的地方出现。

```rust
pub struct Error {
    private: ErrorImpl,
}

enum ErrorImpl {
    Io(io::Error),
    // 即使 other_crate 不是稳定的也没关系，
    // 因为 ErrorImpl 是私有的。
    Dep(other_crate::Error),
}

// 哦不！这将 other_crate 引入了当前 crate 的公共 API。
impl From<other_crate::Error> for Error {
    fn from(err: other_crate::Error) -> Self {
        Error { private: ErrorImpl::Dep(err) }
    }
}
```

## Crate 及其依赖项需具有宽松的许可证 (C-PERMISSIVE)

Rust 项目生成的软件是双重许可的，可以选择 [MIT] 或 [Apache 2.0] 许可证。为了与 Rust 生态系统最大限度地兼容，建议 crate 采用相同的方式。其他选项将在下文描述。

这些 API 指南不详细解释 Rust 的许可证，但在 [Rust FAQ] 中有少量说明。这些指南关注于与 Rust 的互操作性问题，并不全面覆盖许可选项。

[MIT]: https://github.com/rust-lang/rust/blob/master/LICENSE-MIT
[Apache 2.0]: https://github.com/rust-lang/rust/blob/master/LICENSE-APACHE
[Rust FAQ]: https://github.com/dtolnay/rust-faq#why-a-dual-mitasl2-license

要将 Rust 许可证应用于您的项目，请在 `Cargo.toml` 中定义 `license` 字段：

```toml
[package]
name = "..."
version = "..."
authors = ["..."]
license = "MIT OR Apache-2.0"
```

然后在仓库根目录添加文件 `LICENSE-APACHE` 和 `LICENSE-MIT`，其中包含许可证文本（可以从 choosealicense.com 获取，如 [Apache-2.0](https://choosealicense.com/licenses/apache-2.0/) 和 [MIT](https://choosealicense.com/licenses/mit/)）。

在您的 README.md 末尾添加：

```
## License

Licensed under either of

 * Apache License, Version 2.0
   ([LICENSE-APACHE](LICENSE-APACHE) or http://www.apache.org/licenses/LICENSE-2.0)
 * MIT license
   ([LICENSE-MIT](LICENSE-MIT) or http://opensource.org/licenses/MIT)

at your option.

## Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in the work by you, as defined in the Apache-2.0 license, shall be
dual licensed as above, without any additional terms or conditions.
```

除了双重 MIT/Apache-2.0 许可证，Rust crate 作者常用的另一种许可方式是单一宽松许可证，如 MIT 或 BSD。这种许可方案也与 Rust 的许可证完全兼容，因为它施加了 Rust 的 MIT 许可证的最低限制。

为了与 Rust 完美的许可证兼容性，不建议 crate 仅选择 Apache 许可证。尽管 Apache 许可证是宽松的，但它施加了超出 MIT 和 BSD 许可证的限制，可能会在某些情况下阻碍或禁止使用，因此仅有 Apache 许可证的软件在某些情况下无法使用大多数 Rust 运行时栈。

crate 的依赖项的许可证可能会影响 crate 本身的分发限制，因此具有宽松许可证的 crate 通常应仅依赖于具有宽松许可证的 crate。
