# 文档

## 包级文档详尽且包含示例 (C-CRATE-DOC)

请参阅 [RFC 1687](https://github.com/rust-lang/rfcs/pull/1687)。

## 所有项目都有 rustdoc 示例 (C-EXAMPLE)

每个公共模块、特征、结构体、枚举、函数、方法、宏和类型定义都应有一个示例来展示其功能。

此指南应在合理范围内应用。一个项目上的示例链接到另一个项目的相关示例可能就足够了。例如，如果只有一个函数使用特定类型，可以在函数或类型上编写一个示例，并从另一个地方链接到它。

示例的目的不仅仅是展示*如何使用*该项目。读者通常已经了解如何调用函数、匹配枚举和其他基本任务。相反，示例通常旨在展示*为什么有人会想要使用*该项目。

```rust
// 这是一个关于如何使用 clone() 的差劲示例。它机械地展示了如何调用 clone()，但没有说明为什么需要这样做。
fn main() {
    let hello = "hello";

    hello.clone();
}
```

## 示例使用 `?`，而不是 `try!` 或 `unwrap` (C-QUESTION-MARK)

示例代码常被用户直接复制。解包错误应是用户需要做出的有意识的决定。

一个常见的结构化可失败示例代码的方法如下。以 `#` 开头的行在构建示例时由 `cargo test` 编译，但不会出现在用户可见的 rustdoc 中。

```rust
/// ```rust
/// # use std::error::Error;
/// #
/// # fn main() -> Result<(), Box<dyn Error>> {
/// your;
/// example?;
/// code;
/// #
/// #     Ok(())
/// # }
/// ```
```

## 函数文档包括错误、panic 和安全性考虑 (C-FAILURE)

错误条件应在“Errors”部分记录。这也适用于特征方法——如果允许或预期实现返回错误，则应在“Errors”部分记录。

例如，在标准库中，某些 [`std::io::Read::read`] 特征方法的实现可能返回错误。

```rust
/// 从此源中提取一些字节到指定的缓冲区，返回读取的字节数。
///
/// ... 详细信息 ...
///
/// # Errors
///
/// 如果此函数遇到任何形式的 I/O 或其他错误，将返回错误变体。如果返回错误，则必须保证未读取任何字节。
```

panic 条件应在“Panics”部分记录。这也适用于特征方法——如果允许或预期实现发生 panic，则应在“Panics”部分记录。

在标准库中，[`Vec::insert`] 方法可能会 panic。

```rust
/// 在向量中 `index` 位置插入一个元素，将其后的所有元素向右移动。
///
/// # Panics
///
/// 如果 `index` 超出范围，则会 panic。
```

不必记录所有可能的 panic 情况，特别是如果 panic 发生在调用者提供的逻辑中。但当不确定时，宁可记录更多的 panic 情况。

```rust
/// # Panics
///
/// 如果 `T` 的 `Display` 实现 panic，此函数会 panic。
pub fn print<T: Display>(t: T) {
    println!("{}", t.to_string());
}
```

不安全函数应在“Safety”部分记录，解释调用者在正确使用函数时需要保持的所有不变量。

不安全的 [`std::ptr::read`] 要求调用者遵循以下规则。

```rust
/// 从 `src` 读取值而不移动它。这会使 `src` 的内存保持不变。
///
/// # Safety
///
/// 除了接受原始指针外，这也是不安全的，因为它在不防止进一步使用 `src` 的情况下语义上将值移出 `src`。如果 `T` 不是 `Copy`，则必须确保在数据再次被覆盖之前（例如使用 `write`、`zero_memory` 或 `copy_memory`），不要使用 `src` 中的值。请注意，`*src = foo` 算作使用，因为它会尝试删除 `*src` 之前的值。
///
/// 指针必须对齐；如果不是这种情况，请使用 `read_unaligned`。
```

## 文章包含相关内容的超链接 (C-LINK)

可以使用常规 markdown 语法 `[text](url)` 添加内联链接。链接到其他类型可以通过标记 ``[`text`]``，然后在文档字符串末尾添加链接目标 ``[`text`]: <target>``，其中 `<target>` 描述如下。

链接到同一类型中的方法通常如下所示：

```md
[`serialize_struct`]: #method.serialize_struct
```

链接到其他类型通常如下所示：

```md
[`Deserialize`]: trait.Deserialize.html
```

链接目标也可以指向父模块或子模块：

```md
[`Value`]: ../enum.Value.html
[`DeserializeOwned`]: de/trait.DeserializeOwned.html
```

此指南由 RFC 1574 正式推荐，标题为[“链接所有内容”](https://github.com/rust-lang/rfcs/blob/master/text/1574-more-api-documentation-conventions.md#link-all-the-things)。

## Cargo.toml 包含所有常见元数据 (C-METADATA)

`Cargo.toml` 的 `[package]` 部分应包括以下值：

- `authors`
- `description`
- `license`
- `repository`
- `keywords`
- `categories`

此外，还有两个可选的元数据字段：

- `documentation`
- `homepage`

默认情况下，*crates.io* 链接到 [*docs.rs*] 上的 crate 文档。仅当文档托管在 *docs.rs* 以外的地方时，才需要设置 `documentation` 元数据，例如因为 crate 链接到 *docs.rs* 构建环境中不可用的共享库。

[*docs.rs*]: https://docs.rs

仅当 crate 有一个独特的网站而不是源代码库或 API 文档时，才应设置 `homepage` 元数据。不要让 `homepage` 与 `documentation` 或 `repository` 值重复。例如，serde 将 `homepage` 设置为 *https://serde.rs*，这是一个专用网站。

## 发布说明记录所有重大更改 (C-RELNOTES)

crate 的用户可以阅读发布说明，以找到每个已发布版本的更改摘要。应在包级文档和/或 Cargo.toml 中链接的存储库中包含发布说明的链接或说明。

发布说明中应清楚标识[破坏性更改](https://github.com/rust-lang/rfcs/blob/master/text/1105-api-evolution.md)（如 RFC 1105 中定义的）。

如果使用 Git 跟踪 crate 的源代码，则发布到 *crates.io* 的每个版本都应有一个对应的标签标识已发布的提交。对于非 Git VCS 工具，也应使用类似的过程。

```bash
# 标记当前提交
GIT_COMMITTER_DATE=$(git log -n1 --pretty=%aD) git tag -a -m "Release 0.3.0" 0.3.0
git push --tags
```

建议使用带注释的标签，因为如果存在任何带注释的标签，某些 Git 命令会忽略未注释的标签。

### 示例

- [Serde 1.0.0 发布说明](https://github.com/serde-rs/serde/releases/tag/v1.0.0)
- [Serde 0.9.8 发布说明](https://github.com/serde-rs/serde/releases/tag/v0.9.8)
- [Serde 0.9.0 发布说明](https://github.com/serde-rs/serde/releases/tag/v0.9.0)
- [Diesel 变更日志](https://github.com/diesel-rs/diesel/blob/master/CHANGELOG.md)

## Rustdoc 不显示无用的实现细节 (C-HIDDEN)

Rustdoc 应包含用户需要使用 crate 的所有内容，仅此而已。在文中解释相关的实现细节是可以的，但它们不应成为文档中的实际条目。

特别要选择性地显示哪些实现在 rustdoc 中可见——用户需要使用 crate 的所有内容，但没有其他内容。在以下代码中，`PublicError` 的 rustdoc 默认会显示 `From<PrivateError>` 实现。我们选择使用 `#[doc(hidden)]` 隐藏它，因为用户的代码中永远不会有 `PrivateError`，因此该实现对他们来说毫无意义。

```rust
// 此错误类型返回给用户。
pub struct PublicError { /* ... */ }

// 此错误类型由一些私有辅助函数返回。
struct PrivateError { /* ... */ }

// 启用 `?` 运算符的使用。
#[doc(hidden)]
impl From<PrivateError> for PublicError {
    fn from(err: PrivateError) -> PublicError {
        /* ... */
    }
}
```

[`pub(crate)`] 是另一个从公共 API 中删除实现细节的好工具。它允许项目在其自身模块之外使用，但不能在同一个 crate 之外使用。

[`pub(crate)`]: https://github.com/rust-lang/rfcs/blob/master/text/1422-pub-restricted.md