# `macro_rules!`

有了这些知识，我们终于可以引入 `macro_rules!` 了。
如前所述，`macro_rules!` 本身就是一个语法扩展，
也就是说它并不是 Rust 语法的一部分。它的形式如下：

```rust,ignore
macro_rules! $name {
    $rule0 ;
    $rule1 ;
    // …
    $ruleN ;
}
```

*至少得有一条规则* (rule) ，最后一条规则后面的分号可被省略。
规则里你可以使用大/中/小括号：`{}`、`[]`、`()`。
每条“规则”都形如：

```ignore
    ($matcher) => {$expansion}
```

如前所述，分组符号可以是任意一种括号，在模式匹配外侧使用小括号、表达式外侧使用大括号只是出于传统。

注意：选择哪种括号并不会影响宏被调用。
事实上，调用宏时可以使用这三种中任意一种括号，只不过使用 `{ ..  }` 或者 `( ...  );` 
的话会有所不同（关注点在于末尾跟随的分号 `;` ）。
有末尾分号的宏调用 *总是* 会被解析成一个条目 (item)。

如果你好奇的话，`macro_rules!` 的调用将被展开为  *空* (nothing) 。
至少，在 AST 中它被展开为空。
它所影响的是编译器内部的结构，以将该宏注册 (register) 进去。
因此，技术上讲你可以在任何一个空展开合法的位置使用 `macro_rules!` 。

## 模式匹配

当一个宏被调用时，对应的 `macro_rules!` 解释器将按照声明顺序一一检查规则。
对每条规则，它都将尝试将输入标记树的内容与该规则的进行匹配。
某个模式必须与输入 *完全* 匹配才被认为是一次匹配。（这里所译“模式”的原词叫 matcher）

如果输入与某个模式相匹配，则该调用项将被相应的展开内容所取代；
否则，将尝试匹配下条规则。
如果所有规则均匹配失败，则宏展开会失败并报错。

最简单的例子是空模式：

```rust,ignore
macro_rules! four {
    () => { 1 + 3 };
}
```

它将且仅将匹配到空的输入，即 `four!()`、`four![]` 或 `four!{}` 。

注意调用所用的分组标记并 *不需要* 匹配定义时采用的分组标记。
也就是说，你可以通过 `four![]` 调用上述宏，此调用仍将被视作匹配。
只有调用时的输入内容才会被纳入匹配考量范围。

模式中也可以包含字面值 (literal) 标记树，这些标记树必须被完全匹配。
将整个对应标记树在相应位置写下即可。
比如，为匹配标记序列 `4 fn ['spang "whammo"] @_@` ，我们可以这样写：

```rust,ignore
macro_rules! gibberish {
    (4 fn ['spang "whammo"] @_@) => {...};
}
```

使用 `gibberish!(4 fn ['spang "whammo"] @_@'])` 即可成功匹配和调用。

你可以使用能写出的任何标记树。

## 元变量 (Metavariables)

模式 (macther) 还可以包含捕获。
即基于某种通用语法来匹配输入类别，并将结果捕获到元变量中，然后将其替换到输出中。

捕获以美元（`$`）形式写入，其后跟标识符 冒号（`:`），最后是捕获类型，
也称为 **片段分类符** ([fragment-specifier](https://doc.rust-lang.org/nightly/reference/macros-by-example.html#metavariables))，
片段分类符必须是以下类型之一：

* `block` 块：比如用大括号包围起来的语句和/或表达式
* `expr` 表达式 (expression)
* `ident` 标识符 (identifier)：包括关键字 (keywords)
* `item` 条目：比如函数、结构体、模块、`impl` 块
* `lifetime` 生命周期注解：比如 `'foo`、`'static`
* `literal` 字面值：比如 `"Hello World!"`、`3.14`、`'🦀'`
* `meta` 元信息：指 `#[...]` 和 `#![...]` 属性内部的元信息条目
* `pat` 模式 (pattern)
* `path` 路径：比如 `foo`、`::std::mem::replace`、`transmute::<_, int>`
* `stmt` 语句 (statement)
* `tt`：单棵标记树 (single token tree)
* `ty` 类型
* `vis` 可视标识符：可能为空的可视标识符，比如 `pub`、`pub(in crate)`

关于片段分类符更深入的描述请阅读 [片段分类符](./minutiae/fragment-specifiers.md) 章。

比如以下 `macro_rules!` 宏捕获一个表达式输入，并绑定给元变量 `$e`：

```rust,ignore
macro_rules! one_expression {
    ($e:expr) => {...};
}
```

元变量对 Rust 编译器的解析器产生影响，使得元变量总是“正确无误”。
`expr` 表达式元变量总是捕获完整且符合 Rust 编译版本的表达式。

你可以在有限的条件内同时结合字面值标记树和元变量。

当元变量被明确匹配到的时候，只需要写 `$name` 就能引用元变量的值。比如：

```rust,ignore
macro_rules! times_five {
    ($e:expr) => { 5 * $e };
}
```

元变量被替换成完整的 AST 节点，这很像宏展开。这也意味着被 `$e` 捕获的任何标记序列都会被解析成单个完整的表达式。

你也可以一个模式匹配中使用多个元变量：

```rust,ignore
macro_rules! multiply_add {
    ($a:expr, $b:expr, $c:expr) => { $a * ($b + $c) };
}
```

然后在展开的地方（指 `=>` 到 `;` 之间）使用任意多的元变量：

```rust,ignore
macro_rules! discard {
    ($e:expr) => {};
}
macro_rules! repeat {
    ($e:expr) => { $e; $e; $e; };
}
```

有一个特殊的宏变量叫做 [`$crate`] ，它用来指代当前 crate 。

[`$crate`]:./minutiae/hygiene.html#crate

## 反复捕获 (Repetitions)

模式匹配可以重复。这使得匹配一连串标记 (token) 成为可能。反复捕获的一般形式为 `$ ( ...  ) sep rep` 。

（译者注：我更倾向于使用 “反复” 作为动词，使用 “重复“ 作为名词和形容词）

* `$` 是字面上的美元符号标记
* `( ... )` 是被反复匹配的模式，由小括号包围。
* `sep` 是 **可选** 的分隔标记。它不能是括号或者重复操作符。常用例子包括 `,` 和 `;` 。
* `rep` 是 **必须** 的重复操作符。当前可以是：
    * `?`：表示 最多一次重复，所以不能用于分割标记。
    * `*`：表示 零次或多次重复。
    * `+`：表示 一次或多次重复。

重复中可以包含任意有效模式，包括字面标记树、元变量以及任意嵌套的重复。

在展开的地方（指 `=>` 到 `;` 之间），重复也采用相同的语法。

举例来说，下面这个宏将每一个元素都转换成字符串。
它将匹配零或多个由逗号分隔的表达式，并分别将它们拓展成构造 `Vec` 的表达式。

```rust,editable
macro_rules! vec_strs {
    (
        // Start a repetition:
        $(
            // Each repeat must contain an expression...
            $element:expr
        )
        // ...separated by commas...
        ,
        // ...zero or more times.
        *
    ) => {
        // Enclose the expansion in a block so that we can use
        // multiple statements.
        {
            let mut v = Vec::new();

            // Start a repetition:
            $(
                // Each repeat will contain the following statement, with
                // $element replaced with the corresponding expression.
                v.push(format!("{}", $element));
            )*

            v
        }
    };
}

fn main() {
    let s = vec_strs![1, "a", true, 3.14159f32];
    assert_eq!(s, &["1", "a", "true", "3.14159"]);
}
```

你可以在一个重复语句里面使用多次和多个元变量，只要这些元变量以相同的次数重复。
下面的宏和调用代码正常运行：

```rust,editable
macro_rules! repeat_two {
    ($($i:ident)*, $($i2:ident)*) => {
        $( let $i: (); let $i2: (); )*
    }
}

repeat_two!( a b c d e f, u v w x y z );
```

但是这下面的不能运行：

```rust,editable
# macro_rules! repeat_two {
#     ($($i:ident)*, $($i2:ident)*) => {
#         $( let $i: (); let $i2: (); )*
#     }
# }

repeat_two!( a b c d e f, x y z );
```

运行报以下错误：

```
error: meta-variable `i` repeats 6 times, but `i2` repeats 3 times
 --> src/main.rs:6:10
  |
6 |         $( let $i: (); let $i2: (); )*
  |          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

&nbsp;

想了解完整的定义语法，可以参考 Rust Reference 书的
[Macros By Example](https://doc.rust-lang.org/reference/macros-by-example.html#macros-by-example)
一章。
