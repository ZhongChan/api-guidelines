# 未来适应性

## 封装特征防止下游实现 (C-SEALED)

某些特征只应在定义它们的 crate 内实现。使用封装特征模式可以在不破坏现有 API 的情况下更改特征。

```rust
/// 该特征是封装的，无法在此 crate 之外的类型中实现。
pub trait TheTrait: private::Sealed {
    // 用户允许调用的方法。
    fn ...();

    // 用户不允许调用的私有方法。
    #[doc(hidden)]
    fn ...();
}

// 为某些类型实现。
impl TheTrait for usize {
    /* ... */
}

mod private {
    pub trait Sealed {}

    // 仅为这些类型实现。
    impl Sealed for usize {}
}
```

由于私有的 `Sealed` 超特征无法被下游 crate 命名，因此我们保证 `Sealed`（因此 `TheTrait`）的实现仅存在于当前 crate 中。即使添加方法通常会破坏非封装特征的变更，我们也可以在非破坏性发布中向 `TheTrait` 添加方法。此外，我们可以更改未公开记录的方法的签名。

需要注意的是，移除公共方法或更改封装特征中公共方法的签名仍然是破坏性更改。

为了避免用户尝试实现特征时感到沮丧，应该在 rustdoc 中记录该特征是封装的，并且不应在当前 crate 之外实现。

### 示例

- [`serde_json::value::Index`](https://docs.serde.rs/serde_json/value/trait.Index.html)
- [`byteorder::ByteOrder`](https://docs.rs/byteorder/1.1.0/byteorder/trait.ByteOrder.html)

## 结构体具有私有字段 (C-STRUCT-PRIVATE)

将字段设为公共是一个强烈的承诺：它固定了表示选择，并且阻止类型对字段内容进行任何验证或维护任何不变量，因为客户端可以随意修改它。

公共字段最适合 C 风格的 `struct` 类型：复合的、被动的数据结构。否则，考虑提供 getter/setter 方法并隐藏字段。

## 新类型封装实现细节 (C-NEWTYPE-HIDE)

新类型可以用来隐藏表示细节，同时向客户端做出精确的承诺。

例如，考虑一个返回复合迭代器类型的函数 `my_transform`。

```rust
use std::iter::{Enumerate, Skip};

pub fn my_transform<I: Iterator>(input: I) -> Enumerate<Skip<I>> {
    input.skip(3).enumerate()
}
```

我们希望对客户端隐藏此类型，使客户端对返回类型的视图大致为 `Iterator<Item = (usize, T)>`。我们可以使用新类型模式来实现：

```rust
use std::iter::{Enumerate, Skip};

pub struct MyTransformResult<I>(Enumerate<Skip<I>>);

impl<I: Iterator> Iterator for MyTransformResult<I> {
    type Item = (usize, I::Item);

    fn next(&mut self) -> Option<Self::Item> {
        self.0.next()
    }
}

pub fn my_transform<I: Iterator>(input: I) -> MyTransformResult<I> {
    MyTransformResult(input.skip(3).enumerate())
}
```

除了简化签名之外，这种使用新类型的方法允许我们对客户端承诺更少。客户端不知道结果迭代器是如何构造或表示的，这意味着表示可以在未来更改而不会破坏客户端代码。

Rust 1.26 还引入了 [`impl Trait`][] 特性，它比新类型模式更简洁，但有一些额外的权衡，尤其是使用 `impl Trait` 时表达能力有限。例如，返回实现 `Debug` 或 `Clone` 或其他迭代器扩展特性组合的迭代器可能会有问题。总之，`impl Trait` 作为返回类型可能适用于内部 API，甚至可能适合公共 API，但可能并不适用于所有情况。有关更多详细信息，请参阅[Edition Guide 中的“`impl Trait` for returning complex types with ease”][impl-trait-2]部分。

```rust
pub fn my_transform<I: Iterator>(input: I) -> impl Iterator<Item = (usize, I::Item)> {
    input.skip(3).enumerate()
}
```

## 数据结构不重复派生特征边界 (C-STRUCT-BOUNDS)

泛型数据结构不应使用可以派生的特征边界或不增加语义价值的特征边界。`derive` 属性中的每个特征将扩展为一个单独的 `impl` 块，仅适用于实现该特征的泛型参数。

```rust
// 优先选择这样：
#[derive(Clone, Debug, PartialEq)]
struct Good<T> { /* ... */ }

// 而不是这样：
#[derive(Clone, Debug, PartialEq)]
struct Bad<T: Clone + Debug + PartialEq> { /* ... */ }
```

在 `Bad` 上重复派生特征作为边界是不必要的，并且是向后兼容性隐患。考虑在前面的结构上派生 `PartialOrd`：

```rust
// 非破坏性更改：
#[derive(Clone, Debug, PartialEq, PartialOrd)]
struct Good<T> { /* ... */ }

// 破坏性更改：
#[derive(Clone, Debug, PartialEq, PartialOrd)]
struct Bad<T: Clone + Debug + PartialEq + PartialOrd> { /* ... */ }
```

通常来说，向数据结构添加特征边界是破坏性更改，因为该结构的每个消费者都需要开始满足额外的边界。使用 `derive` 属性从标准库派生更多特征不是破坏性更改。

以下特征不应在数据结构的边界中使用：

- `Clone`
- `PartialEq`
- `PartialOrd`
- `Debug`
- `Display`
- `Default`
- `Error`
- `Serialize`
- `Deserialize`
- `DeserializeOwned`

对于其他非派生的特征边界（如 `Read` 或 `Write`），存在灰色地带。它们可能在定义中更好地传达类型的预期行为，但也限制了未来的可扩展性。在数据结构上包含语义上有用的特征边界仍然比包含可派生特征作为边界的问题小。

### 例外

有三种情况需要在结构上使用特征边界：

1. 数据结构引用特征上的关联类型。
2. 边界是 `?Sized`。
3. 数据结构有一个需要特征边界的 `Drop` 实现。Rust 当前要求 `Drop` 实现上的所有特征边界也存在于数据结构上。

### 标准库中的示例

- [`std::borrow::Cow`] 引用 `Borrow` 特征上的关联类型。
- [`std::boxed::Box`] 选择退出隐式 `Sized` 边界。
- [`std::io::BufWriter`] 在其 `Drop` 实现中需要特征边界。