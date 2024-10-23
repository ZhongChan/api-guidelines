# 命名

## 命名风格遵循 RFC 430

基本的 Rust 命名约定在 [RFC 430] 中进行了描述。

一般来说，Rust 对“类型级”构造使用 `UpperCamelCase`（类型和特征），对“值级”构造使用 `snake_case`。更具体地说：

| 项目 | 约定 |
| ---- | ---- |
| Crates | 不明确 |
| Modules | `snake_case` |
| Types | `UpperCamelCase` |
| Traits | `UpperCamelCase` |
| Enum variants | `UpperCamelCase` |
| Functions | `snake_case` |
| Methods | `snake_case` |
| General constructors | `new` 或 `with_more_details` |
| Conversion constructors | `from_some_other_type` |
| Macros | `snake_case!` |
| Local variables | `snake_case` |
| Statics | `SCREAMING_SNAKE_CASE` |
| Constants | `SCREAMING_SNAKE_CASE` |
| Type parameters | 简洁的 `UpperCamelCase`，通常是单个大写字母：`T` |
| Lifetimes | 简短的 `lowercase`，通常是单个字母：`'a`，`'de`，`'src` |
| Features | 不明确 |

在 `UpperCamelCase` 中，缩写和复合词的缩写算作一个词：使用 `Uuid` 而不是 `UUID`，`Usize` 而不是 `USize`，`Stdin` 而不是 `StdIn`。在 `snake_case` 中，缩写小写：`is_xid_start`。

在 `snake_case` 或 `SCREAMING_SNAKE_CASE` 中，一个“词”不应该只由一个字母组成，除非它是最后一个“词”。所以我们有 `btree_map` 而不是 `b_tree_map`，但 `PI_2` 而不是 `PI2`。

Crate 名称不应使用 `-rs` 或 `-rust` 作为后缀或前缀。每个 crate 都是 Rust！没有必要不断提醒用户这一点。

## 临时转换遵循 `as_`、`to_`、`into_` 约定

转换应作为方法提供，其名称前缀如下：

| 前缀 | 成本 | 所有权 |
| ---- | ---- | ----- |
| `as_` | 免费 | 借用 -> 借用 |
| `to_` | 昂贵 | 借用 -> 借用<br>借用 -> 拥有（非 Copy 类型）<br>拥有 -> 拥有（Copy 类型） |
| `into_` | 可变 | 拥有 -> 拥有（非 Copy 类型） |

例如：

- `str::as_bytes()` 提供 `str` 作为 UTF-8 字节切片的视图，这是免费的。输入是借用的 `&str`，输出是借用的 `&[u8]`。
- `Path::to_str` 对操作系统路径的字节执行昂贵的 UTF-8 检查。输入和输出都是借用的。因为此方法在运行时有非平凡的成本，所以不能称为 `as_str`。
- `str::to_lowercase()` 生成 `str` 的 Unicode 正确的小写等价物，这涉及到迭代字符串的字符并可能需要内存分配。输入是借用的 `&str`，输出是拥有的 `String`。
- `f64::to_radians()` 将浮点数从度数转换为弧度。输入是 `f64`。由于 `f64` 复制成本很低，不需要传递引用 `&f64`。称此函数为 `into_radians` 会产生误导，因为输入未被消耗。
- `String::into_bytes()` 提取 `String` 的底层 `Vec<u8>`，这是免费的。它获取 `String` 的所有权并返回拥有的 `Vec<u8>`。
- `BufReader::into_inner()` 获取缓冲读取器的所有权并提取底层读取器，这是免费的。缓冲区中的数据被丢弃。
- `BufWriter::into_inner()` 获取缓冲写入器的所有权并提取底层写入器，这需要潜在昂贵的缓冲数据刷新。

转换前缀 `as_` 和 `into_` 通常减少抽象，或者暴露对底层表示的视图（`as`），或者将数据解构为其底层表示（`into`）。而前缀 `to_` 的转换通常保持在同一抽象级别，但做一些工作以从一种表示转换为另一种表示。

当类型包装单个值以将其与更高级别的语义关联时，访问包装值应通过 `into_inner()` 方法提供。这适用于提供缓冲的包装器，如 `BufReader`，编码或解码，如 `GzDecoder`，原子访问，如 `AtomicBool`，或任何类似语义。

如果转换方法名称中的 `mut` 限定符构成返回类型的一部分，则应按其在类型中出现的方式出现。例如 `Vec::as_mut_slice` 返回一个可变切片；它做了它所说的。这种名称优于 `as_slice_mut`。

## Getter 名称遵循 Rust 约定

在 Rust 代码中，`get_` 前缀通常不用于 getter。

例外情况是，当只有一个明显的东西可以通过 getter 获得时，使用 `get` 命名。例如 `Cell::get` 访问 `Cell` 的内容。

对于运行时验证（例如边界检查）的 getter，考虑添加不安全的 `_unchecked` 变体。通常它们具有以下签名。

```rust
fn get(&self, index: K) -> Option<&V>;
fn get_mut(&mut self, index: K) -> Option<&mut V>;
unsafe fn get_unchecked(&self, index: K) -> &V;
unsafe fn get_unchecked_mut(&mut self, index: K) -> &mut V;
```

getter 和转换之间的区别可能微妙且不总是明确的。例如 `TempDir::path` 可以被理解为临时目录文件系统路径的 getter，而 `TempDir::into_path` 是一种转换，将删除临时目录的责任转移给调用者。由于 `path` 是一个 getter，因此将其称为 `get_path` 或 `as_path` 是不正确的。

## 生成迭代器的方法遵循 `iter`、`iter_mut`、`into_iter`

对于包含类型 `U` 元素的容器，迭代器方法应命名为：

```rust
fn iter(&self) -> Iter             // Iter 实现 Iterator<Item = &U>
fn iter_mut(&mut self) -> IterMut  // IterMut 实现 Iterator<Item = &mut U>
fn into_iter(self) -> IntoIter     // IntoIter 实现 Iterator<Item = U>
```

此指南适用于概念上同质的集合的数据结构。作为反例，`str` 类型是保证为有效 UTF-8 的字节切片。这在概念上比同质集合更复杂，因此它提供 `str::bytes` 以字节迭代和 `str::chars` 以字符迭代，而不是提供 `iter`/`iter_mut`/`into_iter` 组的迭代器方法。

此指南仅适用于方法，而不适用于函数。例如，`url` crate 中的 `percent_encode` 返回一个百分比编码字符串片段的迭代器。使用 `iter`/`iter_mut`/`into_iter` 约定不会带来任何清晰度。

## 迭代器类型名称与生成它们的方法匹配

名为 `into_iter()` 的方法应返回名为 `IntoIter` 的类型，其他返回迭代器的方法同理。

此指南主要适用于方法，但通常也适用于函数。例如，`url` crate 中的 `percent_encode` 函数返回一个迭代器类型，名为 `PercentEncode`。

这些类型名称在其所属模块前缀下最有意义，例如 `vec::IntoIter`。

## 功能名称没有占位词

不要在 [Cargo 功能] 的名称中包含没有意义的词，如 `use-abc` 或 `with-abc`。直接将功能命名为 `abc`。

这最常见于可选依赖于 Rust 标准库的 crate。正确的做法是：

```toml
# 在 Cargo.toml 中

[features]
default = ["std"]
std = []
```

```rust
// 在 lib.rs 中
#![no_std]

#[cfg(feature = "std")]
extern crate std;
```

不要将功能命名为 `use-std` 或 `with-std` 或任何不是 `std` 的创造性名称。此命名约定与 Cargo 为可选依赖推断的隐式功能命名一致。

Cargo 要求功能是可加的，所以像 `no-abc` 这样的负面命名几乎从来都不正确。

## 名称使用一致的词序

以下是标准库中的一些错误类型：

- `JoinPathsError`
- `ParseBoolError`
- `ParseCharError`
- `ParseFloatError`
- `ParseIntError`
- `RecvTimeoutError`
- `StripPrefixError`

所有这些都使用动词-对象-错误的词序。如果我们添加一个表示地址解析失败的错误，为了一致性，我们会希望将其命名为 `ParseAddrError` 而不是 `AddrParseError`。

特定的词序选择并不重要，但要注意 crate 内的一致性以及与标准库中类似功能的一致性。
