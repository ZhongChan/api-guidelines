# Macros

## 输入语法应与输出相呼应 (C-EVOCATIVE)

Rust 宏允许你设计几乎任何输入语法。尽量保持输入语法与用户代码的其他部分一致，通过尽可能地反映现有 Rust 语法来实现这一点。注意关键字和标点符号的选择和位置。

一个好的指导原则是使用与宏输出中产生的语法相似的语法，尤其是关键字和标点符号。

例如，如果你的宏声明了一个具有特定名称的结构体，请在名称前加上关键字 `struct`，以向读者表明正在声明一个具有给定名称的结构体。

```rust
// 优先选择这种方式...
bitflags! {
    struct S: u32 { /* ... */ }
}

// ...而不是没有关键字...
bitflags! {
    S: u32 { /* ... */ }
}

// ...或一些临时的词。
bitflags! {
    flags S: u32 { /* ... */ }
}
```

另一个例子是分号与逗号。Rust 中的常量后跟分号，因此如果你的宏声明了一连串的常量，它们也应该跟分号，即使语法与 Rust 的其他部分略有不同。

```rust
// 普通常量使用分号。
const A: u32 = 0b000001;
const B: u32 = 0b000010;

// 因此优先选择这种方式...
bitflags! {
    struct S: u32 {
        const C = 0b000100;
        const D = 0b001000;
    }
}

// ...而不是这种方式。
bitflags! {
    struct S: u32 {
        const E = 0b010000,
        const F = 0b100000,
    }
}
```

宏是如此多样，以至于这些具体例子可能不相关，但要考虑如何将相同的原则应用于你的情况。

## 项目宏与属性良好结合 (C-MACRO-ATTR)

生成多个输出项目的宏应支持为任一项目添加属性。一个常见的用例是将单个项目放在 `cfg` 后面。

```rust
bitflags! {
    struct Flags: u8 {
        #[cfg(windows)]
        const ControlCenter = 0b001;
        #[cfg(unix)]
        const Terminal = 0b010;
    }
}
```

生成结构体或枚举作为输出的宏应支持属性，以便输出可以与 `derive` 一起使用。

```rust
bitflags! {
    #[derive(Default, Serialize)]
    struct Flags: u8 {
        const ControlCenter = 0b001;
        const Terminal = 0b010;
    }
}
```

## 项目宏可在任何允许项目的地方使用 (C-ANYWHERE)

Rust 允许将项目放在模块级别或更小的范围内，如函数内。项目宏应像普通项目一样在所有这些地方正常工作。测试套件应包括在至少模块范围和函数范围内调用宏。

```rust
#[cfg(test)]
mod tests {
    test_your_macro_in_a!(module);

    #[test]
    fn anywhere() {
        test_your_macro_in_a!(function);
    }
}
```

这是一个简单的例子，说明事情如何出错，这个宏在模块范围内工作正常，但在函数范围内失败。

```rust
macro_rules! broken {
    ($m:ident :: $t:ident) => {
        pub struct $t;
        pub mod $m {
            pub use super::$t;
        }
    }
}

broken!(m::T); // okay, expands to T and m::T

fn g() {
    broken!(m::U); // 编译失败，super::U 指的是包含模块而不是 g
}
```

## 项目宏支持可见性说明符 (C-MACRO-VIS)

遵循 Rust 语法，宏生成的项目默认是私有的，若指定 `pub` 则为公有。

```rust
bitflags! {
    struct PrivateFlags: u8 {
        const A = 0b0001;
        const B = 0b0010;
    }
}

bitflags! {
    pub struct PublicFlags: u8 {
        const C = 0b0100;
        const D = 0b1000;
    }
}
```

## 类型片段应灵活 (C-MACRO-TY)

如果你的宏接受一个类型片段，如输入中的 `$t:ty`，它应该可以用于以下所有情况：

- 基本类型：`u8`, `&str`
- 相对路径：`m::Data`
- 绝对路径：`::base::Data`
- 向上相对路径：`super::Data`
- 泛型：`Vec<String>`

这是一个简单的例子，说明事情如何出错，这个宏在基本类型和绝对路径上工作正常，但在相对路径上失败。

```rust
macro_rules! broken {
    ($m:ident => $t:ty) => {
        pub mod $m {
            pub struct Wrapper($t);
        }
    }
}

broken!(a => u8); // okay

broken!(b => ::std::marker::PhantomData<()>); // okay

struct S;
broken!(c => S); // 编译失败
```