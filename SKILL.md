---
name: wrust
description: >-
    编写符合惯例、健壮、可维护的 Rust。每当编写、审阅、重构、调试或讲解 Rust 代码时，
    都应使用此 skill：包括新的 crate、模块、函数、结构体、trait、错误类型、异步代码、
    unsafe 或 FFI 代码、Cargo.toml、测试或 rustdoc。只要提及 Rust、cargo、crate、
    .rs 文件、借用检查器、生命周期、trait、clippy 或 rustfmt，即使用户没有要求
    “最佳实践”或“惯用”代码，也应触发。综合自 Google Rust 风格指南、Rust API 指南、
    Effective Rust、Rust 设计模式和 Microsoft 实用 Rust 指南。
---

# Rust 最佳实践

提炼自三份 Rust 风格与设计指南，每份指南的摘要均位于 `references/`。
当它们意见不一致时，本文会说明应遵循哪一项以及原因。

## 1. 思维方式

- **编写符合 Rust 形态的代码。** 不要逐项移植 C++、Java、C# 或 Python 构造。
  用 `Result` 建模错误，用所有权和 `Drop` 建模资源，用 trait 和枚举建模多态，
  用 `Option` 建模可空性。带继承意味的技巧（对“基类”结构体使用 `Deref`、
  `Manager`/`Factory` 类型、`throw_if_null` 辅助函数）都是代码异味。
- **让无效状态无法表示。** 将不变量推入类型：用枚举代替 bool 标志和“若 Y 则必须为 X”
  的注释，用 newtype 表示单位和 ID，用可失败构造函数一次性验证。这样编译器就能强制
  执行注释只能提出的要求。
- **应用程序代码不同于库代码。** 库代码（任何存在外部用户的代码）应严格遵循下文的
  API 设计规则。应用程序代码应优先追求简单直接：使用具体类型、`anyhow` 风格错误、
  返回 `Vec`，不做推测性的泛化。
- **让工具完成工作。** 格式交给 rustfmt，惯用法检查交给 clippy，内存安全交给编译器。
  将注意力投入设计。

## 2. 交付代码之前

运行以下命令，并修复它们报告的所有问题：

```bash
cargo fmt
cargo clippy --all-targets --all-features -- -D warnings
cargo doc --no-deps   # 库：捕获失效的文档链接和缺失的文档
```

- 修复 lint，而不是消除它的提示。当某个 lint 对单个项确实不适用时，在范围最小的项上
  使用 `#[expect(clippy::name, reason = "...")]`。当 lint 不再触发时，`expect` 会
  发出 warning，从而清理过期抑制。仅在生成代码或宏输出中使用 `#[allow]`。
- 绝不要在源代码中加入 `#![deny(warnings)]`：新工具链会新增 warning 并破坏构建。
  应在 CI 中使用 `cargo clippy -- -D warnings` 将 warning 视为错误。
- 在 `Cargo.toml`（`[lints]` 或 `[workspace.lints]`）中集中配置 lint，不要散落 crate
  属性。如果项目没有配置，建议使用 https://github.com/morganlinton/BeautifulRust 中的
  `lints/lints.toml` 和 `lints/clippy.toml`；其中的抑制类 lint 需要 `rust-version`
  1.81 或更高版本。

## 3. 所有权与借用

**优先使用唯一所有权。** 只有当共享所有权属于设计的一部分时才使用 `Rc`/`Arc`，
只有当内部可变性属于设计的一部分时才使用 `RefCell`/`Mutex`。它们并非失败，但会将检查
从编译期移到运行时，因此应有意使用。

**不要通过克隆让借用检查器停止抱怨。** 仅为结束借用而存在的克隆会悄然产生副本，
后续更改也会丢失。大致按以下顺序先尝试：

1. 用命名的 `let` 延长临时值的生命周期，或用块缩短借用。
2. 将长表达式拆成带类型的中间值，使失败步骤清晰可见。
3. 使用 `std::mem::take`（要求 `Default`）、`std::mem::replace` 或 `Option::take`
   从 `&mut self` 中移出字段，而不是克隆它。
4. 将大型结构体拆成较小的结构体，使互不重叠的字段可以分别借用。编译器能在单个函数内
   跟踪互不重叠的字段借用，但无法跨方法调用跟踪。
5. 有意克隆小型不可变数据；若原因不明显，应在注释中说明。
6. 最后才考虑使用拥有数据的 `Vec` 加索引，或搭配 `Weak` 反向指针的
   `Rc<RefCell<_>>`。避免自引用结构体。

```rust
// 无需克隆即可从 &mut 枚举中移出值。
fn a_to_b(e: &mut MyEnum) {
    if let MyEnum::A { name, x: 0 } = e {
        *e = MyEnum::B { name: std::mem::take(name) };
    }
}
```

**接受最通用的借用类型。** 使用 `&str` 而不是 `&String`，`&[T]` 而不是 `&Vec<T>`，
`&T` 而不是 `&Box<T>`，`Option<&T>` 而不是 `&Option<T>`。这样，解引用强制转换可让
调用者传入拥有的值、借用或字面量。当只读取路径、字符串或字节缓冲区时，参数可使用
`impl AsRef<Path>` / `impl AsRef<str>` / `impl AsRef<[u8]>`（绝不要用于结构体字段；
字段应存储 `PathBuf` 和 `String`）。只进行迭代的库代码应接受
`impl IntoIterator<Item = T>`；应用程序代码可以直接接受 `&[T]`。

**只在需要所有权时获取所有权，并按值获取。** 如果函数会存储或消耗参数，应接受 `T`，
而不是接受 `&T` 后在内部调用 `.clone()`。只读取时接受 `&T`；原地修改时接受
`&mut T`。以元组或结构体返回多个值，不要通过 `&mut` 输出参数返回（像 `Read::read`
那样复用调用者拥有的缓冲区是例外）。

**结构体优先拥有数据。** 持有 `&'a T` 的结构体会迫使每个容器都带上生命周期。
拥有 `String` 或 `Vec<u8>` 通常更简单，而且很少成为瓶颈。类型确实带生命周期时，
使用描述性名称（`'src`、`'conn`）；简单函数使用 `'_` 和 `'a` 即可。

**准备完成后冻结。** 使用 `let mut` 构建，再以 `let data = data;` 重新绑定，
防止之后的代码意外修改它。

**不要让 `Rc`、`Arc`、`Box` 和 `RefCell` 出现在公共签名中。** 接受 `&T`、`&mut T`
或 `T`，让调用者决定如何拥有数据。必须共享的服务可在内部用 `Arc<Inner>` 包装状态，
并实现低成本 `Clone`。

## 4. 类型与 API 设计

### 命名

| 项                         | 大小写                 | 说明                                           |
| -------------------------- | ---------------------- | ---------------------------------------------- |
| 类型、trait、枚举变体      | `UpperCamelCase`       | 首字母缩略词视为一个单词：`Uuid`、`HttpClient` |
| 函数、方法、模块、局部变量 | `snake_case`           |                                                |
| 常量、静态变量             | `SCREAMING_SNAKE_CASE` |                                                |
| 类型参数                   | 简短，取自主约束       | `T`、`R: Read`、`W: Write`                     |
| 生命周期                   | 小写短名称             | 位于类型上时使用描述性名称                     |

- getter 不加 `get_` 前缀：`fn len(&self)`、`fn name(&self) -> &str`、
  `fn name_mut(&mut self)`。只有存在唯一且显然要获取的内容时才使用裸 `get`
  （`Cell::get`）。相关命名包括：`set_foo(T)`、`take_foo()`、用于闭包访问的
  `with_foo(|f| ..)`，以及受检查访问器对应的 unsafe `_unchecked` 方法。
- 转换前缀表示成本和所有权：`as_`（免费，借用到借用）、`to_`（昂贵，借用到拥有或
  借用到借用）、`into_`（消耗 `self`）。按类型顺序放置 `mut`：使用
  `as_mut_slice`，而不是 `as_slice_mut`。
- 构造函数：`new`（主要）、`with_capacity` / `new_with_foo`（变体）、`try_new`
  （可能失败）、`from_foo`（转换；重要转换应配套 `From` 实现）。I/O 资源使用领域动词：
  `File::open`、`TcpStream::connect`。
- `iter` / `iter_mut` / `into_iter` 的返回类型命名为 `Iter` / `IterMut` / `IntoIter`；
  `keys()` 返回 `Keys`。
- 错误类型按“动词-宾语-error”命名：`ParseBoolError`，而不是 `BoolParseError`。
- 避免含混后缀 `Service`、`Manager`、`Factory`、`Helper`、`Util`。按事物本身命名
  （`Bookings`、`Dispatcher`、`FooBuilder`）。类型名不要重复模块名（使用 `foo::Id`，
  而不是 `foo::FooId`）。避免外部人员不熟悉的缩写。
- 如果实现 `len()`，也要实现 `is_empty()`。
- feature 名称应简洁直接：`std`、`serde`、`alloc`，绝不使用 `use-std` 或 `no-std`。

### 用类型建模领域

- **Newtype** 用于语义区分：`struct Miles(f64)` 和 `struct Kilometers(f64)` 无法混用，
  而类型别名做不到这一点。布局重要时添加 `#[repr(transparent)]`，并 derive 或转发
  所需 trait（`derive_more` 可减少样板代码）。编码不变量的 newtype（`Month`）应在
  可失败构造函数（`TryFrom`、`FromStr`、`try_new`）中验证，并且绝不提供从原始类型
  转换的不会失败的 `From`。
- **公共或共享签名中不使用 `bool` 参数。** 调用点的 `report(true)` 难以理解，也无法
  扩展第三种模式。使用双变体枚举（`Verbosity::Summary`、`Verbosity::Full`）或两个方法
  （`report()`、`report_verbose()`）：使用 `Widget::new(Size::Small, Shape::Round)`，
  而不是 `Widget::new(true, false)`。唯一例外是调用点就在几行之内的私有辅助函数。
  重构时，已有 `bool` 参数是应消除的异味之一，而不是需要保留的细节。
- **命名字段优于元组**，但单字段 newtype 和范围很小的局部代码除外。
- **缺失值使用 `Option`，绝不使用哨兵值**，如 `-1` 或 `""`。使用**携带数据的枚举**，
  而不是“仅当 Y 被设置时字段 X 才有意义”。
- **使用最强的标准库类型：**路径使用 `PathBuf` / `&Path`，时间跨度使用 `Duration`；
  `NonZeroU32` 仅在内部使用（API 边界保持普通整数）。
- **独立标志**使用 `bitflags` crate；枚举表示恰好一个选项。

### 积极 derive 和实现标准 trait

下游 crate 无法为你的类型添加标准库 trait（孤儿规则），因此必须由你实现。每个字段都支持
时应 derive：`Debug`（每个公共类型都应实现；包含机密时手动实现以脱敏）、`Clone`
（独占资源和密钥材料除外）、`Copy`（仅用于小型、可按位复制的类型，因为编译器会悄然
复制）、`PartialEq` / `Eq`、`PartialOrd` / `Ord`（存在全序时，绝不发布没有 `Ord`
的 `PartialOrd`）、`Hash`（与 `Eq` 一致）、`Default`。任何供人阅读的类型都应实现
`Display`：错误类型、类字符串包装器。为纯函数，以及错误和惰性值等丢弃即属 bug 的类型
添加 `#[must_use]`。

实现 `From` / `TryFrom`，绝不直接实现 `Into` / `TryInto`（blanket impl 会自动提供）。
优先使用 `from` / `into`，而不是会悄然允许有损转换的 `as`。实现 `FromStr`，而不是
临时自定义的 `from_str` 方法；集合应实现 `FromIterator` 和 `Extend`，并为拥有和借用形式
都实现 `IntoIterator`。仅在智能指针和拥有数据的集合上实现 `Deref`（如 `Vec<T>` 到
`[T]`），绝不要用它伪造继承。仅按运算符的数学含义重载运算符。

不要在结构体定义上添加可派生的 trait 约束：使用 `#[derive(Clone)] struct Good<T>`，
而不是 `struct Bad<T: Clone + Debug>`。约束应放在真正需要它们的 `impl` 块上。

如果手动实现 `Ord`，也要手动实现与之相符的 `PartialEq`、`PartialOrd` 和 `Hash`
（比较相同字段，忽略相同字段）。在自定义 `Ord` 旁使用派生的 `PartialEq` 会违反标准库
契约，并导致 `BTreeMap` 和 `sort`/`dedup` 行为异常。在文档中说明相等性忽略哪些字段。

### 构造函数、builder 与默认值

- 即使已有 `Default`，也要提供 `new()`，并使二者一致（一个委托给另一个）。derive 或
  实现 `Default`，让调用者可以写 `..Default::default()`；公共字段组成的选项集合
  结构体应始终实现它。
- 最多两个可选参数：使用固有构造函数（`new`、`with_a`、`with_b`）。超过两个，或构造
  过程需要验证或有副作用时，添加 builder：`Foo::builder()` 返回 `FooBuilder`，setter
  以字段命名（`timeout(d)`，而不是 `set_timeout`）且不会失败，
  `build() -> Result<Foo, Error>` 完成包括跨字段检查在内的所有验证。除非 `build` 必须移出值，否则优先使用
  非消耗型 setter（`&mut self -> &mut Self`）。
- 含四个或更多松散参数的构造函数应将参数组合成语义类型：使用
  `Deposit::new(account, amount)`，而不是四个 `&str` / `u64` 参数。

### 可见性与演进

- 默认私有。除非结构体是被动的 C 风格数据，否则字段保持私有。在使用 `pub` 前先考虑
  `pub(crate)` 或 `pub(super)`。以后扩大可见性不属于破坏性变更；反之则是，而且每个
  `pub` 项都是一项承诺。
- 跨 crate 边界的公共枚举和带公共字段结构体应标记为 `#[non_exhaustive]`，除非其集合
  是稳定本体。匹配这类枚举时，列出要处理的变体并添加通配符分支。
- 将不希望下游实现的 trait seal 起来，并在文档中说明：

    ```rust
    pub trait TheTrait: private::Sealed { /* ... */ }
    mod private {
        pub trait Sealed {}
        impl Sealed for usize {}
    }
    ```

- 用 `impl Iterator<Item = T>`（内部 API）或 newtype（必须保持稳定的公共 API）隐藏
  迭代器实现类型。
- 每个公共项只通过一个路径暴露。除非你在编写 umbrella crate，否则不要 `pub use`
  外部 crate 的项。避免向公共 API 泄漏第三方类型；将 `serde` 实现放在名为 `serde`
  的 feature 后。依赖项类型必须出现时，重新导出该依赖项，使调用者能引用匹配版本。
- 仅当函数会一直保持 `const` 时才将其标记为 `const`。不要暴露 `ErrorKind` 枚举或其他
  会固定你本想保持私有的失败模式的内部细节。

### Trait 与泛型

- 仅为泛型编程（`T: Trait`、`impl Trait`、`dyn Trait`）或作为扩展 trait（`FooExt`）
  定义 trait。不要仅为组织代码而使用 trait。将核心功能放在固有方法中，trait 实现转发
  给它们，使调用者无需导入。
- 按以下顺序优先选择：具体类型；泛型（`T: Trait`、`impl Trait`）；`dyn Trait`，
  最好包装在自己的结构体中（`struct DynStore(Arc<dyn Store>)`）。泛型提供静态分派和
  内联；trait 对象支持异构集合和更小的二进制。不要在服务层类型上嵌套泛型
  （`Service<Backend>` 可以；`Service<Backend<Store>>` 不可以）。
- 尽早决定 trait 是否必须对象安全。泛型方法或接收者之外的 `Self` 会破坏对象安全；
  用 `where Self: Sized` 排除这类方法。
- 尽量减少 trait 的必需方法，并像 `Iterator` 从 `next` 推导其他方法那样，为所有可推导
  方法提供默认实现。
- 如果函数有明确接收者，就将其设为方法。没有接收者的计算应是自由函数，而不是
  `Database::check(params)`。关联函数用于构造。
- 闭包：使用能够满足需求的最宽泛约束（调用一次用 `FnOnce`，会修改捕获值用 `FnMut`，
  否则用 `Fn`），优先使用 `Fn*` 约束，而不是裸 `fn` 指针。闭包参数放在最后；
  不要接受多个闭包。
- 在整个 crate 中保持参数顺序一致：调用特有参数在前，无处不在的上下文（logger、
  allocator）在后。
- 只有函数、泛型和 trait 都无法完成任务时才使用宏。优先使用 `macro_rules!` 而不是过程宏，
  让宏输入看起来像它生成的输出，并且绝不让宏改变类型种类或函数签名。

### 重构他人的代码

- 修复请求涉及的每一种异味，而不只挑便宜的：仅为满足借用检查器的克隆、`&String` /
  `&Vec<T>` 参数、可能失败路径上的 `unwrap`、签名中的 `Rc<RefCell<_>>`、`get_` getter、
  `bool` 参数、使用 `+` 拼接字符串、`#![deny(warnings)]`、未实现 `Error` 的错误类型、
  输出中非确定性的 `HashMap` 迭代。
- 除非变更行为本身就是目的，否则保持公共行为不变：相同操作、相同成功与失败语义、
  相同输出字节。将 panic 改为 `Err` 是调用者所期望的变更。删除错误变体、移除 derive，
  或改变构造函数参数则不是；如果签名必须变化，应在变更列表中展示前后版本并解释原因。
- 保留你不理解的特殊行为并将其指出，而不是悄然“修复”。
- 模块没有测试时，先用几个测试固定当前行为，再进行修改。

## 5. 错误与 panic

根据契约选择机制：

| 情形                                       | 使用                                   |
| ------------------------------------------ | -------------------------------------- |
| 缺失是正常情况（`first()`、`checked_add`） | `Option<T>`                            |
| 调用者可能需要知道出了什么问题             | `Result<T, E>`                         |
| 代码内部的 bug 或违反不变量                | `panic!` / `assert!` / `expect("why")` |

**Panic 不是异常。** panic 意味着“立即停止”。绝不要用它报告不良输入，不要依赖
`catch_unwind` 实现控制流（它在 `panic = "abort"` 下不起作用，并可能使状态不一致），
也绝不要在库中捕获 panic 后继续运行。如果提供会 panic 的便捷 API 确有合理性，应同时
提供可失败 API（`get` / `get_unchecked`、`new` / `try_new`），并在 `# Panics` 下说明。

**每个有意的 panic 都应带有包含相关值的消息：**
`assert!(buf.len() >= HEADER, "buffer too small: {} bytes", buf.len())`、
`expect("config is loaded before workers start")`。优先使用 `expect("reason")`，而不是
`unwrap()`。不要在消息中复述条件。

**使用模式匹配，而不是先检查再 unwrap：**使用 `if let Some(v) = opt { .. }` 或
`let Some(v) = opt else { return .. };`，绝不要使用
`if opt.is_some() { opt.unwrap() }`。

**锁中毒是致命的：**使用 `.lock().expect("lock poisoned")`。不要通过 `into_inner`
恢复。

**库错误类型**（公共 `Result` 中的任何 `E`）：

- 实现 `std::error::Error`（它要求 `Debug` + `Display`），满足
  `Send + Sync + 'static`，并为包装的原因实现 `source()`。绝不要将 `()`、`String`
  或 `Box<dyn Error>` 用作库的公共错误类型。
- `Display` 消息使用小写、简洁且末尾无标点：`"invalid header length"`。
- 实现 `From<Upstream>`，让 `?` 完成转换。仅对外部错误或需要添加上下文时使用
  `map_err`。
- 两种合理形态：每种原因一个变体的枚举（`thiserror` 很适合生成这种代码），或每种情形
  一个结构体，内部带私有 `kind`，并提供 `is_xxx()` 和访问器方法。无论哪种方式，都不要
  创建一个列出整个 crate 中所有失败模式的全局枚举，也不要暴露调用者无法处理的细节。
- 如果可失败函数消耗了参数，应将参数放入错误中返回，让调用者无需克隆即可重试
  （`FromUtf8Error::into_bytes`）。
- 绝不要在库的公共 API 中暴露 `anyhow::Error`。

**应用程序错误处理：**`anyhow` 或 `eyre` 搭配 `.context("opening config")` 是正确工具。
应用层只使用一种错误类型，不要使用多种。`main` 可以返回
`Result<(), anyhow::Error>`。

**析构函数绝不失败、panic 或阻塞。** 提供 `close(self) -> Result<()>` 处理可能失败的
拆除，让 `Drop` 只尽力清理。

**记录失败模式：**任何 `Result` 都提供 `# Errors`，任何可达 panic 都提供 `# Panics`，
每个 `unsafe fn` 都提供 `# Safety`。

## 6. 日常惯用法

- 当处理管线较短且可自上而下阅读时，**优先使用迭代器而不是手写循环：**
  `values.iter().filter(..).map(..).collect()`、`.any()`、`.sum()`、`.fold()`。
  将可能失败的步骤收集为 `Result<Vec<_>, E>` 并应用 `?`。循环体较大、承担多项职责，
  或需要别扭的提前返回时，保留显式 `for` 循环。将长链绑定到命名变量，不要直接放在
  `for` 头部；优先使用 `for x in it`，而不是 `.for_each(|x| ..)`。
- 在比 `match` 更清晰时使用 **`Option` / `Result` 组合器**（`map`、`and_then`、
  `ok_or_else`、`unwrap_or_else`、`as_ref`、`as_deref`）；不清晰时使用 `match`。
- 使用带内联参数的 **`format!` / `write!`**（`format!("{name}: {count}")`）进行
  拼接和替换，不要链式使用 `+` 或 `str::replace`。热点路径中使用 `write!` 写入现有缓冲区。
- **当推断会掩盖意图时显式标注类型：**包括 `.into()`、`.parse()`、`.collect()` 和
  复杂迭代器结果。优先使用 `let x: Vec<u32> = ..`，而不是 turbofish。
- **使用 drop guard 完成清理**，确保提前返回或栈展开时也会执行。将其绑定到名称
  （`let _guard = ..`），因为 `let _ = ..` 会立即丢弃。
- **闭包捕获混合所有权时：**在块中重新绑定，再使用 `move`：

    ```rust
    let job = {
        let cfg = cfg.clone();
        move || run(cfg, data)
    };
    ```

- **导入风格：**每个模块路径一条 `use`（`use foo::bar::{Bar, Baz};`）；导入类型和 trait
  后不带限定使用（`Regex`）；自由函数带模块限定（`mem::take`、`fs::read`）。除非整个
  文件都使用某个 trait，否则在需要它的函数内部导入。除测试模块中的 `use super::*`，
  以及匹配枚举的函数内部使用 `use Enum::*` 外，不使用 glob 导入。
- **魔法数字**应改为命名 `const`，并通过文档注释说明值的来源以及更改它会影响什么。
- **日志：**通过 `tracing` 或 `log` 记录结构化事件，不使用 `println!` / `dbg!`；
  CLI 自身的 stdout 输出除外。

## 7. 并发与异步

- 公共类型及其返回的 future 应为 `Send`（通常还应为 `Sync`）。通过编译期检查固定这一点，
  防止重构悄然造成退化：

    ```rust
    const _: () = {
        const fn assert_send<T: Send>() {}
        assert_send::<MyClient>();
    };
    ```

    跨 `.await` 持有 `Rc` 或 `RefCell` 借用会使 future 成为 `!Send`。

- 优先使用消息传递（channel），而不是共享状态。必须共享时：将具有相互关联不变量的数据
  置于同一把锁下，将锁作用域控制在几行内，持锁时绝不调用用户闭包，绝不从公共方法返回
  `MutexGuard`。
- 使用 `async fn`，而不是 `fn -> impl Future`；trait 中，或热点路径必须在 async 块外
  执行准备工作时除外。
- async 中长时间运行的 CPU 密集型工作必须让出执行权（每几十微秒调用一次
  `tokio::task::yield_now().await`），或移至 `spawn_blocking`。
- 库代码不启动原始线程，也不假定运行时；从调用者接受 executor、spawn 函数，或
  `impl AsyncRead` / `impl Read`。

## 8. Unsafe 与 FFI

- 仅因以下三种原因编写 `unsafe`：安全代码无法表达的新颖抽象、经基准验证的性能需求，
  或 FFI 与平台调用。优先使用标准库和经过良好审阅的 crate；其中已经包含审计过的
  `unsafe`。
- 每个 `unsafe` 块或 impl 都应有 `// SAFETY:` 注释，说明不变量为何成立。每个
  `unsafe fn` 都应有 `# Safety` 文档小节，列出调用者必须满足的条件。启用
  `unsafe_op_in_unsafe_fn`。
- 将 `unsafe` 限制在能够维护其不变量的最小模块中，并用安全 API 包装；模块边界就是
  健全性边界。如果一个安全函数能通过任意安全调用序列导致未定义行为，它就是不健全的；
  不健全的代码绝不可接受。
- 仅当误用可能导致未定义行为时才将函数标记为 `unsafe`，不要仅因其危险而标记
  （`delete_database()` 是安全函数）。
- 不要用 `transmute` 转换枚举或规避生命周期，也不要用 `unsafe impl Send` 让编译器
  保持沉默。
- 在 Miri 下并使用对抗性输入测试 unsafe 代码。
- FFI：共享结构体添加 `#[repr(C)]`，使用定长整数类型，字符串使用 `CString` / `CStr`，
  在同一侧分配和释放，用 RAII 类型包装 C 所有的指针并在其 `Drop` 中释放，检查空指针并
  返回 `Result`，通过 `catch_unwind` 阻止 panic 越过边界。在 `foo-sys` crate 中用
  `bindgen` 生成绑定，并将业务逻辑放在安全核心 crate 中。

## 9. Crate、模块与依赖项

- **优先使用小型、单一用途的 crate，**在扁平的 `crates/` 工作区中组织；绝不要把一个
  crate 嵌套到另一个 crate 的 `src/` 中。当 crate 出现第二种用途或第二个子目录时进行
  拆分。通过 `[workspace.dependencies]` 和 `[workspace.lints]` 共享版本与 lint。
- **模块：**叶子模块以主要项命名（`Foo` 位于 `foo.rs`）。非叶子模块只包含 `mod` 声明
  和重新导出。将关键项放在 crate 根目录，其余按使用场景分组（`account`、`network`），
  而不是按种类分组（`traits`、`errors`）。不要定义 prelude。
- **逐项重新导出**（`pub use foo::{A, B};`），绝不使用 glob。
- **依赖项：**每个依赖项都会增加编译时间、二进制大小和供应链暴露面，包括 `build.rs`
  和过程宏。使用范围（`"1"`、`"1.4"`），绝不使用 `"*"` 或不必要的 `"="`。
  二进制项目应提交 `Cargo.lock`。使用 `cargo tree --duplicates`、`cargo deny`、
  `cargo udeps` 和 `cargo audit` 审计。
- **Feature 可叠加且名称简洁：**`std`、`serde`、`alloc`、`test-util`。绝不使用
  `no-std`、`use-serde`，也不要使用会移除或更改公共项的 feature。不要用 feature
  控制结构体字段或 trait 方法。将 mock 和 fake 放在 `test-util` 后。
- **Semver：**破坏性变更包括移除或重命名公共项、向未标记 `#[non_exhaustive]` 的枚举
  添加变体、向可公开构造的结构体添加字段、添加 trait 约束，或使 trait 不再对象安全。
  应先在次版本中弃用，再在主版本中移除。新 crate 使用最新稳定 edition，并有意设置 MSRV。
- 不要在 `main()` 之前运行代码（不使用 `ctor` 风格 crate）。不要在稳定工具链上使用
  不稳定功能。

## 10. 文档与注释

- 每个公共项使用 `///`；每个公共模块和 crate 根文件顶部使用 `//!`。首句应为单独一行、
  约十五个单词，因为它会成为列表中的摘要。
- 适用时按以下顺序提供小节：`# Examples`、`# Errors`、`# Panics`、`# Safety`。
  示例使用 `?`，并用以 `# ` 开头的行隐藏 `fn main`，绝不使用 `unwrap()`，因为用户会
  原样复制。用正文描述参数，不使用表格。
- 记录签名无法表达的内容：前置条件、副作用、所有权转移、复杂度和原因。不要复述类型或
  名称；在 `remove_element` 上写 `/// Removes an element.` 是噪声。
- 使用 ``[`Type`]`` 链接项。用 `#[doc(hidden)]` 隐藏用户永远不会主动触发的 impl
  （`From<PrivateError> for PublicError`）。
- `//` 注释解释原因，而不是行为：`// SAFETY:`、`// TODO(issue-123):`、不明显的魔法值、
  带链接的变通方案。代码文档中不要包含设计日志、“为何选择 X”的文章或规则合规表。
  记录最终状态。

## 11. 测试

- 私有内部实现的单元测试放在 `#[cfg(test)] mod tests { use super::*; .. }` 中。
  任何可通过公共 API 访问的内容都在 `tests/` 中编写集成测试；两种位置都适用时优先后者。
  文档示例也是测试。
- 测试可观察行为和性质（单调性、往返转换、幂等性），不要测试复述的常量或镜像的分支。
- 修复 bug 前先写一个失败测试。覆盖每个 feature 组合和每条平台特有路径。
- 任何涉及 I/O、时间、随机性或网络的内容都必须可模拟：在 `test-util` 后提供
  `new_mocked() -> (Self, MockCtrl)`，不要提供会执行 I/O 的 `Default`。
- 使用 `cargo fuzz` 模糊测试不可信输入的解析器。使用 `criterion` 或 `divan` 和
  `std::hint::black_box` 进行基准测试。用 Miri 运行 `unsafe` 代码。
- 断言应带上下文，使失败信息易读：`assert_eq!(got, want, "input {input:?}")`。

## 12. 性能优化顺序

首先为易用性设计，测量后再优化。优化时依次考虑：使用 `Vec::with_capacity` 或
`collect()`，而不是 push 循环；在热点 API 中复用调用者拥有的缓冲区
（`fn read_into(&mut self, buf: &mut Vec<u8>)`）；长期存在的不可变序列使用
`Box<[T]>`、`Box<str>`、`Arc<str>`；对未预留容量构建且长期存在的集合调用
`shrink_to_fit`；仅对可信键使用更快的哈希器（`foldhash`、`FxHash`）；批量处理，
而不是逐项加锁；应用程序可设置 `mimalloc` 等 `#[global_allocator]`。小型普通类型应
derive `Copy`，大型类型绝不要这样做。

## 13. 最终检查清单

返回 Rust 代码前确认：

- `cargo fmt`、`cargo clippy -- -D warnings` 和 `cargo test` 均无问题。
- 测试和 `main` 外没有 `unwrap()`；每个 `expect`、`assert` 和 `panic` 都有包含相关值
  的消息。
- 公共函数接受 `&str`、`&[T]`、`impl AsRef<_>` 或 `impl IntoIterator`，而不是
  `&String` 或 `&Vec<T>`。
- 每个公共类型都 derive `Debug` 和适用的常用 trait；错误类型实现
  `Error + Display + Send + Sync + 'static`。
- 公共签名中没有 `bool` 参数；没有不配套 `PartialEq` 和 `Hash` 的手写 `Ord`。
- 没有唯一作用是满足借用检查器的 `.clone()`。
- 没有 glob 导入、prelude、glob 重新导出或泄漏的第三方类型。
- 每个 `unsafe` 块都有 `// SAFETY:` 注释；每个 `unsafe fn` 都有 `# Safety` 文档。
- 公共项都有包含单行摘要的文档，并在适用时提供 `# Errors` / `# Panics`。
- 应用程序代码保持简单；库代码遵循 API 规则。

## 参考资料

需要了解规则背后的理由或任务的完整检查清单时，请阅读以下文档。每份文档都是一个来源的
浓缩摘要。

- `references/rust-api-guidelines.md`（https://rust-lang.github.io/api-guidelines/）：
  设计公共 API 或 crate；涵盖全部 54 个检查项及约定。
- `references/effective-rust.md`（https://www.lurklurk.org/effective-rust/）：在不同方案间
  进行选择：泛型与 `dyn`、`Result` 与 panic、迭代器与循环、借用检查器应对方法、依赖项、CI。
- `references/rust-design-patterns.md`（https://rust-unofficial.github.io/patterns/）：
  带代码的惯用法、模式和反模式。
- `references/microsoft-rust-guidelines.md`（https://microsoft.github.io/rust-guidelines/）：
  错误结构体设计、panic 策略、unsafe 策略、async、工作区布局、性能。
- `references/google-rust-style-guide.md`（https://google.github.io/styleguide/rustguide.html）：
  格式、导入、模块布局、访问器命名、应用程序与库的区别。
