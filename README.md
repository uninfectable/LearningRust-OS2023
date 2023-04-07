# LearningRust-OS2023
# rust learn

## 所有权

### 栈与堆

栈中的所有数据都必须占用已知且固定的大小。在编译时大小未知或大小可能变化的数据，要改为存储在堆上。 

堆是缺乏组织的：当向堆放入数据时，你要请求一定大小的空间。内存分配器（memory allocator）在堆的某处找到一块足够大的空位，把它标记为已使用，并返回一个表示该位置地址的 指针（pointer）

**（将数据推入栈中并不被认为是分配）**。因为指向放入堆中数据的指针是已知的并且大小是固定的，你可以将该指针存储在栈上，不过当需要实际数据时，必须访问指针。

当你的代码调用一个函数时，传递给函数的值（包括可能指向堆上数据的指针）和函数的局部变量被压入栈中。当函数结束时，这些值被移出栈。

### 所有权规则

Rust 中的每一个值都有一个 所有者（owner）。
值在任一时刻有且只有一个所有者。
当所有者（变量）离开作用域，这个值将被丢弃。

当 s 进入作用域 时，它就是有效的。
这一直持续到它 离开作用域 为止。

### string类型

前面介绍的类型都是已知大小的，可以存储在栈中，并且当离开作用域时被移出栈，如果代码的另一部分需要在不同的作用域中使用相同的值，可以快速简单地复制它们来创建一个新的独立实例。

**字符串字面值**：被硬编码进程序里的字符串值 不适合使用文本的每一种场景 它们是不可变的 并非所有字符串的值都能在编写代码时就知道

可以使用 `from` 函数基于字符串字面值来创建 `String`

```rust
let s = String::from("hello");
```

两个冒号 `::` 是运算符，允许将特定的 `from` 函数置于 `String` 类型的命名空间（namespace）下

#### 区别:内存与分配

字符串字面值 我们在编译时就知道其内容，所以文本被直接硬编码进最终的可执行文件中 快速且高效

`String` 类型 在堆上分配一块在编译时未知大小的内存来存放内容。

1. 必须在运行时向内存分配器（memory allocator）请求内存。
2. 需要一个当我们处理完 `String` 时将内存返回给分配器的方法。

第一部分由我们完成：当调用 `String::from` 时，它的实现 (*implementation*) 请求其所需的内存。这在编程语言中是非常通用的。

第二部分实现各有区别。 **垃圾回收**（*garbage collector*，*GC*）的语言中，GC 自行记录并清除不再使用的内存。在大部分没有 GC 的语言中，识别出不再使用的内存并调用代码显式释放是我们的责任，跟请求内存的时候一样。忘记回收了会浪费内存。过早回收了，将会出现无效变量。重复回收，会产生 bug。我们需要精确的为一个 `allocate` 配对一个 `free`。

Rust 采取了一个不同的策略：内存在拥有它的变量离开作用域后就被自动释放。

当变量离开作用域，Rust 为我们调用一个特殊的函数。这个函数叫做 [`drop`](https://doc.rust-lang.org/std/ops/trait.Drop.html#tymethod.drop)，在这里 `String` 的作者可以放置释放内存的代码。Rust 在结尾的 `}` 处自动调用 `drop`。

> 注意：在 C++ 中，这种 item 在生命周期结束时释放资源的模式有时被称作 **资源获取即初始化**（*Resource Acquisition Is Initialization (RAII)*）。如果你使用过 RAII 模式的话应该对 Rust 的 `drop` 函数并不陌生。

这个模式在更复杂的场景下代码的行为可能是不可预测的。

#### 变量与数据交互的方式（一）：移动

在 Rust 中，多个变量可以采取不同的方式与同一数据进行交互。

```rust
let x = 5;
let y = x;
```

“将 `5` 绑定到 `x`；接着生成一个值 `x` 的拷贝并绑定到 `y`”。两个变量是有已知固定大小的简单值，所以这两个 `5` 被放入了栈中。

 `String` 版本：

```rust
let s1 = String::from("hello");
let s2 = s1;
```

`String` 由三部分组成，如图左侧所示：一个指向存放字符串内容内存的指针，一个长度，和一个容量。这一组数据存储在栈上。右侧则是堆上存放内容的内存部分。

![Two tables: the first table contains the representation of s1 on the stack, consisting of its length (5), capacity (5), and a pointer to the first value in the second table. The second table contains the representation of the string data on the heap, byte by byte.](C:\Users\1\Desktop\rustlings\trpl04-01.svg)

长度表示 `String` 的内容当前使用了多少字节的内存。容量是 `String` 从分配器总共获取了多少字节的内存。

当我们将 `s1` 赋值给 `s2`，`String` 的数据被复制了，这意味着我们从栈上拷贝了它的指针、长度和容量。我们并没有复制指针指向的堆上数据。换句话说，内存中数据的表现如下图所示。

![Three tables: tables s1 and s2 representing those strings on the stack, respectively, and both pointing to the same string data on the heap.](C:\Users\1\Desktop\rustlings\trpl04-02.svg)

这个表现形式看起来 **并不像** 图 4-3 中的那样，如果 Rust 也拷贝了堆上的数据，那么内存看起来就是这样的。如果 Rust 这么做了，那么操作 `s2 = s1` 在堆上数据比较大的时候会对运行时性能造成非常大的影响。

![Four tables: two tables representing the stack data for s1 and s2, and each points to its own copy of string data on the heap.](C:\Users\1\Desktop\rustlings\trpl04-03.svg)

当变量离开作用域后，Rust 自动调用 `drop` 函数并清理变量的堆内存。图 2 展示了两个数据指针指向了同一位置。这就有了一个问题：当 `s2` 和 `s1` 离开作用域，他们都会尝试释放相同的内存。这是 **二次释放**（*double free*）的错误，也是内存安全性 bug 之一。两次释放（相同）内存会导致内存污染，它可能会导致潜在的安全漏洞。

为了确保内存安全，在 `let s2 = s1;` 之后，Rust 认为 `s1` 不再有效，这段代码不能运行：

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;

    println!("{}, world!", s1);
}
```

如果你在其他语言中听说过术语 **浅拷贝**（*shallow copy*）和 **深拷贝**（*deep copy*），那么拷贝指针、长度和容量而不拷贝数据可能听起来像浅拷贝。不过因为 Rust 同时使第一个变量无效了，这个操作被称为 **移动**（*move*），而不是叫做浅拷贝。上面的例子可以解读为 `s1` 被 **移动** 到了 `s2` 中。那么具体发生了什么，如图 4-4 所示。

![Three tables: tables s1 and s2 representing those strings on the stack, respectively, and both pointing to the same string data on the heap. Table s1 is grayed out be-cause s1 is no longer valid; only s2 can be used to access the heap data.](https://kaisery.github.io/trpl-zh-cn/img/trpl04-04.svg)

这里还隐含了一个设计选择：**Rust 永远也不会自动创建数据的 “深拷贝”**。任何 **自动** 的复制可以被认为对运行时性能影响较小。

#### 变量与数据交互的方式（二）：克隆

如果我们 **确实** 需要深度复制 `String` 中堆上的数据，而不仅仅是栈上的数据，可以使用一个叫做 `clone` 的通用函数。

**只在栈上的数据：拷贝**

原因是像整型这样的在编译时已知大小的类型被整个存储在栈上，所以拷贝其实际的值是快速的。这意味着没有理由在创建变量 `y` 后使 `x` 无效。换句话说，这里没有深浅拷贝的区别，所以这里调用 `clone` 并不会与通常的浅拷贝有什么不同。

Rust 有一个叫做 `Copy` trait 的特殊注解，可以用在类似整型这样的存储在栈上的类型上。如果一个类型实现了 `Copy` trait，那么一个旧的变量在将其赋值给其他变量后仍然可用。

Rust 不允许自身或其任何部分实现了 `Drop` trait 的类型使用 `Copy` trait。如果我们对其值离开作用域时需要特殊处理的类型使用 `Copy` 注解，将会出现一个编译时错误。

任何一组简单标量值的组合都可以实现 `Copy`，任何不需要分配内存或某种形式资源的类型都可以实现 `Copy` 。

### 所有权与函数

将值传递给函数与给变量赋值的原理相似。向函数传递值可能会移动或者复制，就像赋值语句一样。

```rust
fn main() {
    let s = String::from("hello");  // s 进入作用域

    takes_ownership(s);             // s 的值移动到函数里 ...
                                    // ... 所以到这里不再有效

    let x = 5;                      // x 进入作用域

    makes_copy(x);                  // x 应该移动函数里，
                                    // 但 i32 是 Copy 的，
                                    // 所以在后面可继续使用 x

} // 这里，x 先移出了作用域，然后是 s。但因为 s 的值已被移走，
  // 没有特殊之处

fn takes_ownership(some_string: String) { // some_string 进入作用域
    println!("{}", some_string);
} // 这里，some_string 移出作用域并调用 `drop` 方法。
  // 占用的内存被释放

fn makes_copy(some_integer: i32) { // some_integer 进入作用域
    println!("{}", some_integer);
} // 这里，some_integer 移出作用域。没有特殊之处

```

变量的所有权总是遵循相同的模式：将值赋给另一个变量时移动它。当持有堆中数据值的变量离开作用域时，其值将通过 `drop` 被清理掉，除非数据被移动为另一个变量所有。

可以使用元组来返回多个值

```rust
fn main() {
    let s1 = String::from("hello");

    let (s2, len) = calculate_length(s1);

    println!("The length of '{}' is {}.", s2, len);
}

fn calculate_length(s: String) -> (String, usize) {
    let length = s.len(); // len() 返回字符串的长度

    (s, length)
}

```

## 引用与借用

**引用**（*reference*）像一个指针，因为它是一个地址，我们可以由此访问储存于该地址的属于其他变量的数据。 与指针不同，引用确保指向某个特定类型的有效值。

变量 `s` 有效的作用域与函数参数的作用域一样，不过当 `s` 停止使用时并不丢弃引用指向的数据，因为 `s` 并没有所有权。当函数使用引用而不是实际值作为参数，无需返回值来交还所有权，因为就不曾拥有所有权。

**可变引用**允许我们修改一个借用的值

```rust
fn main() {
    let mut s = String::from("hello");

    change(&mut s);
}

fn change(some_string: &mut String) {
    some_string.push_str(", world");
}

```

首先，我们必须将 `s` 改为 `mut`。然后在调用 `change` 函数的地方创建一个可变引用 `&mut s`，并更新函数签名以接受一个可变引用 `some_string: &mut String`。这就非常清楚地表明，`change` 函数将改变它所借用的值。

可变引用有一个很大的限制：如果你有一个对该变量的可变引用，你就不能再创建对该变量的引用。

这一限制以一种非常小心谨慎的方式允许可变性，防止同一时间对同一数据存在多个可变引用。

这个限制的好处是 Rust 可以在编译时就避免数据竞争。

不能在拥有不可变引用的同时拥有可变引用，然而，多个不可变引用是可以的