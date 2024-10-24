# 可靠性

## 函数验证其参数 (C-VALIDATE)

Rust API 通常不遵循[健壮性原则]：“发送时要保守；接收时要宽容”。

相反，Rust 代码应尽可能_强制_输入的有效性。

可以通过以下机制实现强制（按优先顺序列出）。

### 静态验证

选择一种排除无效输入的参数类型。

例如，优先选择

```rust
fn foo(a: Ascii) { /* ... */ }
```

而不是

```rust
fn foo(a: u8) { /* ... */ }
```

其中 `Ascii` 是 `u8` 的_包装器_，保证最高位为零；有关创建类型安全包装器的更多信息，请参见新类型模式 ([C-NEWTYPE])。

静态验证通常几乎没有运行时成本：它将成本推到边界（例如，当 `u8` 首次转换为 `Ascii` 时）。它还可以在编译期间而不是通过运行时失败来早期捕获错误。

另一方面，有些属性很难或无法用类型表达。

### 动态验证

在处理输入时（或必要时提前）验证输入。动态检查通常比静态检查更容易实现，但有几个缺点：

1. 运行时开销（除非检查可以作为处理输入的一部分进行）。
2. 错误检测延迟。
3. 引入失败情况，通过 `panic!` 或 `Result`/`Option` 类型，这些必须由客户端代码处理。

#### 使用 `debug_assert!` 的动态验证

与动态验证相同，但可以轻松关闭生产构建的昂贵检查。

#### 带有选择退出的动态验证

与动态验证相同，但添加了选择退出检查的兄弟函数。

惯例是用后缀 `_unchecked` 标记这些选择退出函数，或将它们放在 `raw` 子模块中。

在以下情况下可以谨慎使用未经检查的函数：（1）性能要求避免检查，（2）客户端确信输入有效。

## 析构函数永不失败 (C-DTOR-FAIL)

析构函数在 panic 时执行，在这种情况下，失败的析构函数会导致程序中止。

与其在析构函数中失败，不如提供一个单独的方法来检查是否正常拆解，例如返回 `Result` 以标记问题的 `close` 方法。如果未调用该 `close` 方法，则 `Drop` 实现应进行拆解并忽略或记录/跟踪其产生的任何错误。

## 可能阻塞的析构函数有替代方案 (C-DTOR-BLOCK)

同样，析构函数不应调用阻塞操作，这会使调试变得更加困难。考虑提供一个单独的方法来准备无错的非阻塞拆解。