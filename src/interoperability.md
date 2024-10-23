# 互操作性

## 常见特征的实现 (C-COMMON-TRAITS)

Rust 的 trait 系统不允许孤立实现：每个 `impl` 必须位于定义特征或实现类型的 crate 中。因此，定义新类型的 crate 应尽可能实现所有适用的常见特征。

### 关键特征

- `Copy`
- `Clone`
- `Eq`
- `PartialEq`
- `Ord`
- `PartialOrd`
- `Hash`
- `Debug`
- `Display`
- `Default`

通常，类型会同时实现 `Default` 和一个空的 `new` 构造函数。

## 使用标准转换特征 (C-CONV-TRAITS)

应该实现以下转换特征：

- `From`
- `TryFrom`
- `AsRef`
- `AsMut`

不应实现以下转换特征，因为它们已经有基于 `From` 和 `TryFrom` 的 blanket impl：

- `Into`
- `TryInto`

### 标准库示例

- `From<u16>` 实现为 `u32`。
- `TryFrom<u32>` 实现为 `u16`，以应对可能的溢出。

## 集合实现 `FromIterator` 和 `Extend` (C-COLLECT)

实现 `FromIterator` 和 `Extend` 以便集合可以方便地与迭代器方法一起使用，如 `collect`、`partition` 和 `unzip`。

### 标准库示例

- `Vec<T>` 实现了 `FromIterator<T>` 和 `Extend<T>`。

## 数据结构实现 Serde 的 `Serialize` 和 `Deserialize` (C-SERDE)

数据结构类型应实现 `Serialize` 和 `Deserialize`。如果不依赖 Serde，可以通过 Cargo 配置选项来实现。

### 示例

```rust
#[cfg_attr(feature = "serde", derive(Serialize, Deserialize))]
pub struct T { /* ... */ }
```

## 类型尽可能实现 `Send` 和 `Sync` (C-SEND-SYNC)

`Send` 和 `Sync` 是自动实现的。确保类型的线程安全特性准确反映 `Send` 和 `Sync` 状态。

### 测试示例

```rust
#[test]
fn test_send() {
    fn assert_send<T: Send>() {}
    assert_send::<MyStrangeType>();
}

#[test]
fn test_sync() {
    fn assert_sync<T: Sync>() {}
    assert_sync::<MyStrangeType>();
}
```

## 错误类型应有意义且行为良好 (C-GOOD-ERR)

错误类型应实现 `std::error::Error`，并实现 `Send` 和 `Sync`。不要使用 `()` 作为错误类型。

### 示例

```rust
#[derive(Debug)]
struct DoError;

impl Display for DoError { /* ... */ }
impl Error for DoError { /* ... */ }
```

## 二进制数类型提供 `Hex`、`Octal`、`Binary` 格式化 (C-NUM-FMT)

实现以下格式化特征：

- `UpperHex`
- `LowerHex`
- `Octal`
- `Binary`

适用于可以进行位操作的数值类型。

## 泛型读写函数按值接受 `R: Read` 和 `W: Write` (C-RW-VALUE)

标准库的实现允许通过值接受的函数使用可变引用。文档中应提醒用户可以传递可变引用。

### 示例

- `flate2::read::GzDecoder::new`
- `serde_json::from_reader`

这些准则有助于确保 Rust 代码的互操作性和可扩展性。
