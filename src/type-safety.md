# 类型安全

## 新类型提供静态区分 (C-NEWTYPE)

新类型可以在静态上区分底层类型的不同解释。

例如，一个 `f64` 值可能用于表示以英里或公里为单位的数量。使用新类型，我们可以跟踪预期的解释：

```rust
struct Miles(pub f64);
struct Kilometers(pub f64);

impl Miles {
    fn to_kilometers(self) -> Kilometers { /* ... */ }
}
impl Kilometers {
    fn to_miles(self) -> Miles { /* ... */ }
}
```

一旦我们分离了这两种类型，就可以在静态上确保它们不会混淆。例如，函数

```rust
fn are_we_there_yet(distance_travelled: Miles) -> bool { /* ... */ }
```

不能意外地用 `Kilometers` 值调用。编译器会提醒我们进行转换，从而避免某些[灾难性错误]。

[灾难性错误]: http://en.wikipedia.org/wiki/Mars_Climate_Orbiter

## 参数通过类型传达意义，而不是 `bool` 或 `Option` (C-CUSTOM-TYPE)

优先选择

```rust
let w = Widget::new(Small, Round)
```

而不是

```rust
let w = Widget::new(true, false)
```

核心类型如 `bool`、`u8` 和 `Option` 有许多可能的解释。

使用明确的类型（无论是枚举、结构体还是元组）来传达解释和不变量。在上述示例中，如果不查找参数名称，很难立即理解 `true` 和 `false` 所传达的内容，但 `Small` 和 `Round` 更具提示性。

使用自定义类型使以后扩展选项更容易，例如通过添加 `ExtraLarge` 变体。

参见新类型模式 ([C-NEWTYPE])，这是一种以无成本方式用不同名称包装现有类型的方法。

[C-NEWTYPE]: #c-newtype

## 标志集的类型是 `bitflags`，而不是枚举 (C-BITFLAG)

Rust 支持具有显式指定判别值的 `enum` 类型：

```rust
enum Color {
    Red = 0xff0000,
    Green = 0x00ff00,
    Blue = 0x0000ff,
}
```

当 `enum` 类型需要序列化为与其他系统/语言兼容的整数值时，自定义判别值很有用。它们支持“类型安全” API：通过接受一个 `Color`，而不是整数，函数保证获得格式良好的输入，即使稍后将这些输入视为整数。

`enum` 允许 API 从多个选项中请求一个选择。有时，API 的输入是标志集的存在或不存在。在 C 代码中，这通常通过将每个标志对应于特定位来完成，从而允许单个整数表示 32 或 64 个标志。Rust 的 [`bitflags`] crate 提供了这种模式的类型安全表示。

[`bitflags`]: https://github.com/bitflags/bitflags

```rust
use bitflags::bitflags;

bitflags! {
    struct Flags: u32 {
        const FLAG_A = 0b00000001;
        const FLAG_B = 0b00000010;
        const FLAG_C = 0b00000100;
    }
}

fn f(settings: Flags) {
    if settings.contains(Flags::FLAG_A) {
        println!("doing thing A");
    }
    if settings.contains(Flags::FLAG_B) {
        println!("doing thing B");
    }
    if settings.contains(Flags::FLAG_C) {
        println!("doing thing C");
    }
}

fn main() {
    f(Flags::FLAG_A | Flags::FLAG_C);
}
```

## 构建器支持复杂值的构造 (C-BUILDER)

由于构造需要：

* 大量输入
* 复合数据（例如切片）
* 可选配置数据
* 多种选择

某些数据结构构造起来比较复杂，这很容易导致大量不同的构造函数，每个都有许多参数。

如果 `T` 是这样的数据结构，考虑引入一个 `T` _构建器_：

1. 引入一个单独的数据类型 `TBuilder` 用于增量配置 `T` 值。尽可能选择一个更好的名称：例如 [`Command`] 是 [子进程] 的构建器，[`Url`] 可以由 [`ParseOptions`] 创建。
2. 构建器构造函数应仅接受构造 `T` 所需的数据作为参数。
3. 构建器应提供一套方便的方法进行配置，包括逐步设置复合输入（如切片）。这些方法应返回 `self` 以允许链式调用。
4. 构建器应提供一个或多个用于实际构建 `T` 的“终端”方法。

[`Command`]: https://doc.rust-lang.org/std/process/struct.Command.html
[子进程]: https://doc.rust-lang.org/std/process/struct.Child.html
[`Url`]: https://docs.rs/url/1.4.0/url/struct.Url.html
[`ParseOptions`]: https://docs.rs/url/1.4.0/url/struct.ParseOptions.html

当构建 `T` 涉及副作用（如生成任务或启动进程）时，构建器模式尤其适用。

在 Rust 中，构建器模式有两种变体，区别在于对所有权的处理，如下所述。

### 非消耗性构建器（首选）

在某些情况下，构造最终的 `T` 不需要消耗构建器本身。以下是 [`std::process::Command`] 的一个示例：

[`std::process::Command`]: https://doc.rust-lang.org/std/process/struct.Command.html

```rust
// 注意：实际的 Command API 不使用拥有的字符串；这是一个简化版本。

pub struct Command {
    program: String,
    args: Vec<String>,
    cwd: Option<String>,
    // 等等
}

impl Command {
    pub fn new(program: String) -> Command {
        Command {
            program: program,
            args: Vec::new(),
            cwd: None,
        }
    }

    /// 添加传递给程序的参数。
    pub fn arg(&mut self, arg: String) -> &mut Command {
        self.args.push(arg);
        self
    }

    /// 添加多个传递给程序的参数。
    pub fn args(&mut self, args: &[String]) -> &mut Command {
        self.args.extend_from_slice(args);
        self
    }

    /// 设置子进程的工作目录。
    pub fn current_dir(&mut self, dir: String) -> &mut Command {
        self.cwd = Some(dir);
        self
    }

    /// 将命令作为子进程执行，并返回该子进程。
    pub fn spawn(&self) -> io::Result<Child> {
        /* ... */
    }
}
```

注意 `spawn` 方法，它实际上使用构建器配置来生成进程，接受构建器的共享引用。这是可能的，因为生成进程不需要配置数据的所有权。

因为终端 `spawn` 方法只需要一个引用，所以配置方法接受并返回对 `self` 的可变借用。

#### 优点

通过在整个过程中使用借用，`Command` 可以方便地用于单行和更复杂的构造：

```rust
// 单行
Command::new("/bin/cat").arg("file.txt").spawn();

// 复杂配置
let mut cmd = Command::new("/bin/ls");
if size_sorted {
    cmd.arg("-S");
}
cmd.arg(".");
cmd.spawn();
```

### 消耗性构建器

有时构建器必须在构造最终类型 `T` 时转移所有权，这意味着终端方法必须接受 `self` 而不是 `&self`。

```rust
impl TaskBuilder {
    /// 命名待建任务。
    pub fn named(mut self, name: String) -> TaskBuilder {
        self.name = Some(name);
        self
    }

    /// 重定向任务本地 stdout。
    pub fn stdout(mut self, stdout: Box<io::Write + Send>) -> TaskBuilder {
        self.stdout = Some(stdout);
        self
    }

    /// 创建并执行一个新的子任务。
    pub fn spawn<F>(self, f: F) where F: FnOnce() + Send {
        /* ... */
    }
}
```

在这里，`stdout` 配置涉及传递一个 `io::Write` 的所有权，该所有权必须在构建时（在 `spawn` 中）转移到任务。

当构建器的终端方法需要所有权时，有一个基本的权衡：

* 如果其他构建器方法接受/返回可变借用，复杂配置情况将运作良好，但单行配置变得不可能。

* 如果其他构建器方法接受/返回拥有的 `self`，单行继续运作良好，但复杂配置不太方便。

在使简单事情简单，复杂事情可能的原则下，消耗性构建器的所有构建器方法应接受并返回拥有的 `self`。然后客户端代码如下所示：

```rust
// 单行
TaskBuilder::new("my_task").spawn(|| { /* ... */ });

// 复杂配置
let mut task = TaskBuilder::new();
task = task.named("my_task_2"); // 必须重新赋值以保留所有权
if reroute {
    task = task.stdout(mywriter);
}
task.spawn(|| { /* ... */ });
```

单行与以前一样运作，因为所有权通过每个构建器方法传递，直到被 `spawn` 消耗。然而，复杂配置更为冗长：每一步都需要重新赋值构建器。
