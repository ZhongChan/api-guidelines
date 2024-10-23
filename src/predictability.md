# 可预测性

## 智能指针不添加固有方法 (C-SMART-PTR)

例如，这就是 [`Box::into_raw`] 函数定义方式的原因。

[`Box::into_raw`]: https://doc.rust-lang.org/std/boxed/struct.Box.html#method.into_raw

```rust
impl<T> Box<T> where T: ?Sized {
    fn into_raw(b: Box<T>) -> *mut T { /* ... */ }
}

let boxed_str: Box<str> = /* ... */;
let ptr = Box::into_raw(boxed_str);
```

如果将其定义为固有方法，在调用时可能会混淆到底是在调用 `Box<T>` 上的方法还是 `T` 上的方法。

```rust
impl<T> Box<T> where T: ?Sized {
    // 不要这样做。
    fn into_raw(self) -> *mut T { /* ... */ }
}

let boxed_str: Box<str> = /* ... */;

// 这是通过智能指针 Deref 实现访问的 str 上的方法。
boxed_str.chars()

// 这是 Box<str> 上的方法...?
boxed_str.into_raw()
```

## 转换存在于涉及的最具体类型上 (C-CONV-SPECIFIC)

在不确定时，优先使用 `to_`/`as_`/`into_` 而不是 `from_`，因为它们更易于使用（且可以与其他方法链式调用）。

对于许多类型之间的转换，其中一个类型显然更“具体”：它提供了一些附加的不变量或解释，而另一个类型没有。例如，[`str`] 比 `&[u8]` 更具体，因为它是一个 UTF-8 编码的字节序列。

[`str`]: https://doc.rust-lang.org/std/primitive.str.html

转换应存在于涉及的更具体的类型中。因此，`str` 提供了将 `&[u8]` 转换为和从 `&[u8]` 转换的 [`as_bytes`] 方法和 [`from_utf8`] 构造函数。除了直观，这种约定避免了用无尽的转换方法污染具体类型如 `&[u8]`。

[`as_bytes`]: https://doc.rust-lang.org/std/primitive.str.html#method.as_bytes
[`from_utf8`]: https://doc.rust-lang.org/std/str/fn.from_utf8.html

## 有明确接收者的函数是方法 (C-METHOD)

对于与特定类型明确关联的任何操作，优先选择：

```rust
impl Foo {
    pub fn frob(&self, w: widget) { /* ... */ }
}
```

而不是：

```rust
pub fn frob(foo: &Foo, w: widget) { /* ... */ }
```

方法相较于函数有许多优势：

* 它们不需要导入或限定即可使用：您只需要一个合适类型的值。
* 它们的调用会自动借用（包括可变借用）。
* 它们使得回答“我可以用类型 `T` 的值做什么”这个问题变得容易（尤其在使用 rustdoc 时）。
* 它们提供 `self` 表示法，更简洁且通常更清楚地传达所有权区别。

## 函数不使用输出参数 (C-NO-OUT)

对于返回多个 `Bar` 值，优先选择：

```rust
fn foo() -> (Bar, Bar)
```

而不是：

```rust
fn foo(output: &mut Bar) -> Bar
```

复合返回类型如元组和结构体编译效率高且不需要堆分配。如果函数需要返回多个值，应通过这些类型之一来实现。

主要例外：有时函数旨在修改调用者已拥有的数据，例如重用缓冲区：

```rust
fn read(&mut self, buf: &mut [u8]) -> io::Result<usize>
```

## 操作符重载不令人惊讶 (C-OVERLOAD)

通过实现 [`std::ops`] 中的特征，可以为类型提供带有内置语法的操作符（`*`，`|` 等）。这些操作符有很强的预期：仅为与乘法有某种相似性并共享预期属性（例如结合性）的操作实现 `Mul`，其他特征同理。

[`std::ops`]: https://doc.rust-lang.org/std/ops/index.html#traits

## 仅智能指针实现 `Deref` 和 `DerefMut` (C-DEREF)

`Deref` 特征在许多情况下由编译器隐式使用，并与方法解析交互。相关规则专为适应智能指针而设计，因此这些特征应仅用于此目的。

### 标准库中的示例

* [`Box<T>`](https://doc.rust-lang.org/std/boxed/struct.Box.html)
* [`String`](https://doc.rust-lang.org/std/string/struct.String.html) 是指向 [`str`](https://doc.rust-lang.org/std/primitive.str.html) 的智能指针
* [`Rc<T>`](https://doc.rust-lang.org/std/rc/struct.Rc.html)
* [`Arc<T>`](https://doc.rust-lang.org/std/sync/struct.Arc.html)
* [`Cow<'a, T>`](https://doc.rust-lang.org/std/borrow/enum.Cow.html)

## 构造函数是静态的固有方法 (C-CTOR)

在 Rust 中，“构造函数”只是一个约定。关于构造函数命名有多种约定，区别通常很微妙。

构造函数最基本的形式是无参数的 `new` 方法。

```rust
impl<T> Example<T> {
    pub fn new() -> Example<T> { /* ... */ }
}
```

构造函数是为其构造的类型的静态（无 `self`）固有方法。结合完全导入类型名称的做法，这种约定导致信息丰富但简洁的构造：

```rust
use example::Example;

// 构造一个新的 Example。
let ex = Example::new();
```

通常 `new` 应用作实例化类型的主要方法。有时它不带参数，如上述示例。有时它确实带参数，如 [`Box::new`]，它传入要放入 `Box` 的值。

某些类型的构造函数，尤其是 I/O 资源类型，使用不同的命名约定，如 [`File::open`]、[`Mmap::open`]、[`TcpStream::connect`] 和 [`UdpSocket::bind`]。在这些情况下，名称是根据领域选择的。

通常有多种方式构造一个类型。在这些情况下，次要构造函数通常后缀为 `_with_foo`，如 [`Mmap::open_with_offset`]。如果您的类型有多种构造选项，请考虑使用构建者模式 ([C-BUILDER])。

一些构造函数是“转换构造函数”，它们从不同类型的现有值创建新类型。这些通常以 `from_` 开头命名，如 [`std::io::Error::from_raw_os_error`]。注意 `From` 特征 ([C-CONV-TRAITS])，它非常相似。`from_` 前缀的转换构造函数与 `From<T>` 实现之间有三个区别：

* `from_` 构造函数可以是不安全的；而 `From` 实现不能。例如 [`Box::from_raw`]。
* `from_` 构造函数可以接受附加参数以消除源数据的歧义，如 [`u64::from_str_radix`]。
* `From` 实现仅在源数据类型足以确定输出数据类型的编码时适用。当输入只是一个位袋时，如 [`u64::from_be`] 或 [`String::from_utf8`]，转换构造函数名称能够识别它们的含义。

[`Box::from_raw`]: https://doc.rust-lang.org/std/boxed/struct.Box.html#method.from_raw
[`u64::from_str_radix`]: https://doc.rust-lang.org/std/primitive.u64.html#method.from_str_radix
[`u64::from_be`]: https://doc.rust-lang.org/std/primitive.u64.html#method.from_be
[`String::from_utf8`]: https://doc.rust-lang.org/std/string/struct.String.html#method.from_utf8

注意，类型通常会同时实现 `Default` 和 `new` 构造函数。对于同时具有这两者的类型，它们应具有相同的行为。可以在其中一个的基础上实现另一个。

[C-BUILDER]: type-safety.html#c-builder
[C-CONV-TRAITS]: interoperability.html#c-conv-traits

### 标准库中的示例

* [`std::io::Error::new`] 是常用的 IO 错误构造函数。
* [`std::io::Error::from_raw_os_error`] 是基于从操作系统接收到的错误代码的转换构造函数。
* [`Box::new`] 创建一个新的容器类型，接受单个参数。
* [`File::open`] 打开一个文件资源。
* [`Mmap::open_with_offset`] 打开一个内存映射文件，具有附加选项。

[`File::open`]: https://doc.rust-lang.org/stable/std/fs/struct.File.html#method.open
[`Mmap::open`]: https://docs.rs/memmap/0.5.2/memmap/struct.Mmap.html#method.open
[`Mmap::open_with_offset`]: https://docs.rs/memmap/0.5.2/memmap/struct.Mmap.html#method.open_with_offset
[`TcpStream::connect`]: https://doc.rust-lang.org/stable/std/net/struct.TcpStream.html#method.connect
[`UdpSocket::bind`]: https://doc.rust-lang.org/stable/std/net/struct.UdpSocket.html#method.bind
[`std::io::Error::new`]: https://doc.rust-lang.org/std/io/struct.Error.html#method.new
[`std::io::Error::from_raw_os_error`]: https://doc.rust-lang.org/std/io/struct.Error.html#method.from_raw_os_error
[`Box::new`]: https://doc.rust-lang.org/stable/std/boxed/struct.Box.html#method.new
