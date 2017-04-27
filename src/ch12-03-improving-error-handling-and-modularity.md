## 重构改进模块性和错误处理

> [ch12-03-improving-error-handling-and-modularity.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch12-03-improving-error-handling-and-modularity.md)
> <br>
> commit b8e4fcbf289b82c12121b282747ce05180afb1fb

为了改善我们的程序这里有四个问题需要修复，而且他们都与程序的组织方式和如何处理潜在错误有关。

第一，`main`现在进行了两个任务：它解析了参数并打开了文件。对于一个这样的小函数，这并不是一个大问题。然而如果`main`中的功能持续增加，`main`函数处理的单独的任务也会增加。当函数承担了更多责任，它就更难以推导，更难以测试，并且更难以在不破坏其他部分的情况下做出修改。最好能分离出功能这样每个函数就负责一个任务。

这同时也关系到第二个问题：`search`和`filename`是程序中的配置变量，而像`f`和`contents`则用来执行程序逻辑。随着`main`函数的增长，就需要引入更多的变量到作用域中，而当作用域中有更多的变量时，将更难以追踪每个变量的目的。最好能将将配置变量组织进一个结构这样就能使他们的目的更明确了。

第三个问题是如果打开文件失败我们使用`expect`来打印出错误信息，不过这个错误信息只是说`file not found`。除了缺少文件之外还有很多打开文件可能失败的方式：例如，文件可能存在，不过可能没有打开它的权限。如果我们现在就出于这种情况，打印出的`file not found`错误信息就给了用户一个不符合事实的建议！

第四，我们不停的使用`expect`来处理不同的错误，如果用户没有指定足够的参数来运行程序，他们会从 Rust 得到 "index out of bounds" 错误而这并不能明确的解释问题。如果所有的错误处理都位于一处这样将来的维护者在需要修改错误处理逻辑时就只需要咨询一处代码。将所有的错误处理都放在一处也有助于确保我们打印的错误信息对终端用户来说是有意义的。

让我们通过重构项目来解决这些问题。

### 二进制项目的关注分离

`main`函数负责多个任务的组织问题在许多二进制项目中很常见。所以 Rust 社区开发出一个类在`main`函数开始变得庞大时进行二进制程序的关注分离的指导性过程。这些过程有如下步骤：

1. 将程序拆分成 *main.rs* 和 *lib.rs* 并将程序的逻辑放入 *lib.rs* 中。
2. 当命令行解析逻辑比较小时，可以保留在 *main.rs* 中。
3. 当命令行解析开始变得复杂时，也同样将其从 *main.rs* 提取到 *lib.rs*中。
4. 经过这些过程之后保留在`main`函数中的责任是：
    * 使用参数值调用命令行解析逻辑
    * 设置任何其他的配置
    * 调用 *lib.rs* 中的`run`函数
    * 如果`run`返回错误，则处理这个错误

这个模式的一切就是为了关注分离：*main.rs* 处理程序运行，而 *lib.rs* 处理所有的真正的任务逻辑。因为不能直接测试`main`函数，这个结构通过将所有的程序逻辑移动到 *lib.rs* 的函数中使得我们可以测试他们。仅仅保留在 *main.rs* 中的代码将足够小以便阅读就可以验证其正确性。


<!--Since main is already handling the parsing of arguments, why do we need to
add a new function for it, can you say how that improves things? -->
<!-- Sorry, the steps we had were unclear. We've tried rewording. /Carol -->

### 提取参数解析器

首先，我们将提取解析参数的功能。列表 12-5 中展示了新`main`函数的开头，它调用了新函数`parse_config`。目前它仍将定义在 *src/main.rs* 中：

<span class="filename">Filename: src/main.rs</span>

```rust,ignore
fn main() {
    let args: Vec<String> = env::args().collect();

    let (query, filename) = parse_config(&args);

    // ...snip...
}

fn parse_config(args: &[String]) -> (&str, &str) {
    let query = &args[1];
    let filename = &args[2];

    (query, filename)
}
```

<span class="caption">Listing 12-5: Extract a `parse_config` function from
`main`</span>

<!-- Will add ghosting and wingdings in libreoffice /Carol -->

我们仍然将命令行参数收集进一个 vector，不过不同于在`main`函数中将索引 1 的参数值赋值给变量`query`和将索引 2 的值赋值给变量`filename`，我们将整个 vector 传递给`parse_config`函数。接着`parse_config`函数将包含知道哪个参数该放入哪个变量的逻辑，并将这些值返回到`main`。仍然在`main`中创建变量`query`和`filename`，不过`main`不再负责处理命令行参数与变量如何对应。

这对我们这小程序可能有点大材小用，不过我们将采用小的、增量的步骤进行重构。在做出这些改变之后，再次运行程序并验证参数解析是否仍然正常。经常验证你的进展是一个好习惯，这样在遇到问题时能帮助你定位问题的成因。

### 组合配置值

我们可以采取另一个小的步骤来进一步改善这个函数。现在函数返回一个元组，不过立刻又就将元组拆成了独立的部分。这是一个我们可能没有进行正确抽象的信号。

另一个表明还有改进空间的迹象是`parse_config`的`config`部分，它暗示了我们返回的两个值是相关的并都是一个配置值的一部分。目前除了将这两个值组合进元组之外并没有表达这个数据结构的意义：我们可以将这两个值放入一个结构体并给每个字段一个有意义的名字。这会让未来的维护者更容易理解不同的值如何相互关联以及他们的目的。

<!-- above -- I'm not sure why this is a problem --- because they aren't
currently bound together? And why does it imply that -->



> 注意：一些同学将这种拒绝使用相对而言更为合适的复合类型而使用基本类型的模式称为**基本类型偏执**（*primitive obsession*）。

<!-- Ah, I see, so the problems here stem from using simple types to do tasks
inefficiently, when a more complex task could handle it in ways that improve...
behavior? Readability? Can you say as much? -->
<!-- I've tried to clarify above. Note that when Rust programmers talk about
"efficiency", they usually mean "run-time performance", whereas here we're
talking about code design and maintainability and not addressing performance
at all. /Carol -->

列表 12-6 展示了新定义的结构体`Config`，它有字段`query`和`filename`。我们也改变了`parse_config`函数来返回一个`Config`结构体的实例，并更新`main`来使用结构体字段而不是单独的变量：

<span class="filename">Filename: src/main.rs</span>

```rust,should_panic
# use std::env;
# use std::fs::File;
#
fn main() {
    let args: Vec<String> = env::args().collect();

    let config = parse_config(&args);

    println!("Searching for {}", config.query);
    println!("In file {}", config.filename);

    let mut f = File::open(config.filename).expect("file not found");

    // ...snip...
}

struct Config {
    query: String,
    filename: String,
}

fn parse_config(args: &[String]) -> Config {
    let query = args[1].clone();
    let filename = args[2].clone();

    Config {
        query: query,
        filename: filename,
    }
}
```

Listing 12-6: Refactoring `parse_config` to return an instance of a `Config`
struct

<!-- Will add ghosting and wingdings in libreoffice /Carol -->

`parse_config`的签名现在表明它返回一个`Config`值。在`parse_config`的函数体中，之前返回了`args`中`String`值引用的字符串 slice，现在我们选择定义`Config`来使用拥有所有权的`String`值。`main`中的`args`变量是参数值的所有者并只允许`parse_config`函数借用他们，这意味着如果`Config`尝试获取`args`中值的所有权将违反 Rust 的借用规则。

还有许多不同的方式可以处理`String`的数据，而最简单但有些不太高效的方式是调用这些值的`clone`方法。这会生成`Config`实例可以拥有的数据的完整拷贝，不过会比储存字符串数据的引用消耗更多的时间和内存。不过拷贝数据使得代码显得更加直白因为无需管理引用的生命周期，所以在这种情况下牺牲一小部分性能来换取简洁性的取舍是值得的。

<!-- PROD: START BOX -->

> #### 使用`clone`权衡取舍
>
> 由于其运行时消耗，许多 Rustacean 之间有一个趋势是倾向于避免使用`clone`来解决所有权问题。在关于迭代器的第十三章中，我们将会学习如何更有效率的处理这种情况，不过现在，复制一些字符串来取得进展是没有问题的，因为只会进行一次这样的拷贝，而且文件名和要搜索的字符串都比较短。在第一轮编写时拥有一个可以工作但有点低效的程序要比尝试过度优化代码更好一些。随着你对 Rust 更加熟练，将能更轻松的直奔合适的方法，不过现在调用`clone`是完全可以接受的。

<!-- PROD: END BOX -->

我们更新`main`将`parse_config`返回的`Config`实例放入变量`config`中，并更新之前分别使用`search`和`filename`变量的代码为现在的使用`Config`结构体的字段。

现在代码更明确的表现了我们的意图，`query`和`filename`是相关联的并且他们的目的是配置程序如何工作的。任何使用这些值的代码就知道在`config`实例中对应目的的字段名中寻找他们。

### 创建一个`Config`构造函数

<!-- Can you lay out what we intend to do in this section? I wasn't sure even
at the end what we did and why --- why did we create it as parse_config to then
change it to new? -->
<!-- We're making small, incremental changes. In addition to being good
software development practice, we were hoping that by changing one thing at a
time, the process of improving code's design would be easier to follow rather
than just jumping to the best solution. We extracted code into a function, then
it was clearer that we should introduce a struct, then it was clear that the
function we extracted is really a constructor of `Config` and should be written
as such. This refactoring process should be familiar to software developers.
I've tried to add a little recap to the start of this section, I hope that
helps. /Carol -->

目前为止，我们将负责解析命令行参数的逻辑从`main`提取到了`parse_config`函数中，这帮助我们看清值`query`和`filename`是相互关联的并应该在代码中表现这种关系。接着我们增加了`Config`结构体来命名`query`和`filename`的相关目的，并能够从`parse_config`函数中将这些值的名称作为结构体字段名称返回。

所以现在`parse_config`函数的目的是创建一个`Config`实例，我们可以将`parse_config`从一个普通函数变为一个叫做`new`的与结构体关联的函数。做出这个改变使得代码更符合习惯：可以像标准库中的`String`调用`String::new`来创建一个该类型的实例那样，将`parse_config`变为一个与`Config`关联的`new`函数。列表 12-7 展示了需要做出的修改：

<span class="filename">Filename: src/main.rs</span>

```rust,should_panic
# use std::env;
#
fn main() {
    let args: Vec<String> = env::args().collect();

    let config = Config::new(&args);

    // ...snip...
}

# struct Config {
#     query: String,
#     filename: String,
# }
#
// ...snip...

impl Config {
    fn new(args: &[String]) -> Config {
        let query = args[1].clone();
        let filename = args[2].clone();

        Config {
            query: query,
            filename: filename,
        }
    }
}
```

<span class="caption">Listing 12-7: Changing `parse_config` into
`Config::new`</span>

<!-- Will add ghosting and wingdings in libreoffice /Carol -->

这里将`main`中调用`parse_config`的地方更新为调用`Config::new`。我们将`parse_config`的名字改为`new`并将其移动到`impl`块中，这使得`new`函数与`Config`相关联。再次尝试编译并确保它可以工作。

### 修复错误处理

现在我们开始修复错误处理。回忆一下之前提到过如果`args` vector 包含少于 3 个项并尝试访问 vector 中索引 1 或 索引 2 的值会造成程序 panic。尝试不带任何参数运行程序；这将看起来像这样：

```
$ cargo run
    Finished debug [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/greprs`
thread 'main' panicked at 'index out of bounds: the len is 1
but the index is 1',  /stable-dist-rustc/build/src/libcollections/vec.rs:1307
note: Run with `RUST_BACKTRACE=1` for a backtrace.
```

`index out of bounds: the len is 1 but the index is 1`是一个针对程序员的错误信息，然而这并不能真正帮助终端用户理解发生了什么和他们应该做什么。现在就让我们修复它吧。

### 改善错误信息

在列表 12-8 中，在`new`函数中增加了一个检查在访问索引 1 和 2 之前检查 slice 是否足够长。如果 slice 不够长，我们使用一个更好的错误信息 panic 而不是`index out of bounds`信息：

<span class="filename">Filename: src/main.rs</span>

```rust,ignore
// ...snip...
fn new(args: &[String]) -> Config {
    if args.len() < 3 {
        panic!("not enough arguments");
    }
    // ...snip...
```

<span class="caption">Listing 12-8: Adding a check for the number of
arguments</span>

<!-- Will add ghosting and wingdings in libreoffice /Carol -->

这类似于列表 9-8 中的`Guess::new`函数，那里如果`value`参数超出了有效值的范围就调用`panic!`。不同于检查值的范围，这里检查`args`的长度至少是 3，而函数的剩余部分则可以假设这个条件成立的基础上运行。如果
`args`少于 3 个项，这个条件将为真，并调用`panic!`立即终止程序。

有了`new`中这几行额外的代码，再次不带任何参数运行程序并看看现在错误看起来像什么：

```
$ cargo run
    Finished debug [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/greprs`
thread 'main' panicked at 'not enough arguments', src/main.rs:29
note: Run with `RUST_BACKTRACE=1` for a backtrace.
```

这个输出就好多了，现在有了一个合理的错误信息。然而，我们还有一堆额外的信息不希望提供给用户。所以在这里使用列表 9-8 中的技术可能不是最好的；无论如何`panic!`调用更适合程序问题而不是使用问题，正如第九章所讲到的。相反我们可以使用那一章学习的另一个技术：返回一个可以表明成功或错误的`Result`。

<!-- Below -- how does using new fix this, can you lay that our up front? -->
<!-- I'm not sure what you mean, we're already using `new` and the fix continues
to use `new`... /Carol -->

#### 从`new`中返回`Result`而不是调用`panic!`

我们可以选择返回一个`Result`值，它在成功时会包含一个`Config`的实例，而在错误时会描述问题。当`Config::new`与`main`交流时，在使用`Result`类型存在问题时可以使用 Rust 的信号方式。接着修改`main`将`Err`成员转换为对用户更友好的错误，而不是`panic!`调用产生的关于`thread 'main'`和`RUST_BACKTRACE`的文本。

列表 12-9 展示了`Config::new`返回值和函数体中返回`Result`所需的改变：

<span class="filename">Filename: src/main.rs</span>

```rust,ignore
impl Config {
    fn new(args: &[String]) -> Result<Config, &'static str> {
        if args.len() < 3 {
            return Err("not enough arguments");
        }

        let query = args[1].clone();
        let filename = args[2].clone();

        Ok(Config {
            query: query,
            filename: filename,
        })
    }
}
```

<span class="caption">Listing 12-9: Return a `Result` from `Config::new`</span>

<!-- Will add ghosting and wingdings in libreoffice /Carol -->

<!-- what does returning a Result rather than a Config do? -->
<!-- This is what Chapter 9 was about, I've added a few more references
to that chapter to reinforce the connection /Carol -->

现在`new`函数返回一个`Result`，在成功时带有一个`Config`实例而在出现错误时带有一个`&'static str`。回忆一下第十章“静态声明周期”中讲到`&'static str`是一个字符串字面值，也是目前的错误信息。

`new`函数体中有两处修改：当没有足够参数时不再调用`panic!`，而是返回`Err`值。同时我们将`Config`返回值包装进`Ok`成员中。这些修改使得函数符合其新的类型签名。

通过让`Config::new`返回一个`Err`值，这就允许`main`函数处理`new`函数返回的`Result`值并在出现错误的情况更明确的结束进程。

### `Config::new`调用并处理错误

为了处理错误情况并打印一个对用户友好的信息，我们需要像列表 12-10 那样更新`main`函数来处理现在`Config::new`返回的`Result`。另外还需要实现一些`panic!`替我们处理的问题：使用错误码 1 退出命令行工具。非零的退出状态是一个告诉调用程序的进程我们的程序以错误状态退出的惯例信号。

<span class="filename">Filename: src/main.rs</span>

```rust,ignore
use std::process;

fn main() {
    let args: Vec<String> = env::args().collect();

    let config = Config::new(&args).unwrap_or_else(|err| {
        println!("Problem parsing arguments: {}", err);
        process::exit(1);
    });

    // ...snip...
```

<span class="caption">Listing 12-10: Exiting with an error code if creating a
new `Config` fails</span>

<!-- Will add ghosting and wingdings in libreoffice /Carol -->

<!-- In the `main` function itself, we'll handle the `Result` value returned
from the `new` function and exit the process in a cleaner way if `Config::new`
returns an `Err` value.-->
<!-- I moved this line above to the previous section, it seems to at least
partially answer some of my earlier confusions, though I'm not following this
as well as I'd like so not sure if I have this right, can you confirm either
way whether that move makes sense? -->
<!-- That's fine /Carol -->

在上面的列表中，使用了一个之前没有涉及到的方法：`unwrap_or_else`，它定义于标准库的`Result<T, E>`上。使用`unwrap_or_else`可以进行一些自定义的非`panic!`的错误处理。当`Result`是`Ok`时，这个方法的行为类似于`unwrap`：它返回`Ok`内部封装的值。然而，当`Result`是`Err`时，它调用一个**闭包**（*closure*），也就是一个我们定义的作为参数传递给`unwrap_or_else`的匿名函数。第十三章会更详细的介绍闭包。现在你需要理解的是`unwrap_or_else`会将`Err`的内部值，也就是列表 12-9 中增加的`not enough arguments`静态字符串的情况，传递给闭包中位于两道竖线间的参数`err`。闭包中的代码在其运行时可以使用这个`err`值。

<!--Can you give a high-level idea of what the closure does with it? -->
<!-- Does with what? I've tried to elaborate in the above and below paragraphs,
but I'm not sure exactly what's confusing /Carol -->

我们新增了一个`use`行来从标准库中导入`process`。在错误的情况闭包中将被运行的代码只有两行：我们打印出了`err`值，接着调用了`std::process::exit`（在开头增加了新的`use`行从标准库中导入了`process`）。`process::exit`会立即停止程序并将传递给它的数字作为返回状态码。这类似于列表 12-8 中使用的基于`panic!`的错误处理，除了不会在得到所有的额外输出了。让我们试试：

```
$ cargo run
   Compiling greprs v0.1.0 (file:///projects/greprs)
    Finished debug [unoptimized + debuginfo] target(s) in 0.48 secs
     Running `target/debug/greprs`
Problem parsing arguments: not enough arguments
```

非常好！现在输出对于用户来说就友好多了。

### 提取`run`函数

现在我们完成了配置解析的重构：让我们转向程序的逻辑。正如“二进制项目的关注分离”部分的讨论所留下的过程，我们将提取一个叫做`run`的函数来存放目前`main`函数中不属于设置配置或处理错误的所有逻辑。一旦完成这些，`main`函数将简明的足以通过观察来验证，而我们将能够为所有其他逻辑编写测试。

<!-- it contains ALL the function from main? Can you say why we're doing this,
hw this improves it? What is the run function doing? I'm afraid I feel a bit in
the dark here-->
<!-- This is the pattern that we explained in the Separation of Concerns for
Binary Projects section. I've added a reference back to that and reiterated
some of the reasoning from there, but this section isn't introducing the
concept of the `run` function holding the logic that was in `main` /Carol -->

列表 12-11 展示了提取出来的`run`函数。目前我们只进行小的增量式的提取函数的改进并仍将在 *src/main.rs* 中定义这个函数：

<span class="filename">Filename: src/main.rs</span>

```rust,ignore
fn main() {
    // ...snip...

    println!("Searching for {}", config.query);
    println!("In file {}", config.filename);

    run(config);
}

fn run(config: Config) {
    let mut f = File::open(config.filename).expect("file not found");

    let mut contents = String::new();
    f.read_to_string(&mut contents).expect("something went wrong reading the file");

    println!("With text:\n{}", contents);
}

// ...snip...
```

<span class="caption">Listing 12-11: Extracting a `run` function containing the
rest of the program logic</span>

<!-- Will add ghosting and wingdings in libreoffice /Carol -->

现在`run`函数包含了`main`中从读取文件开始的剩余的所有逻辑。`run`函数获取一个`Config`实例作为参数。

#### 从`run`函数中返回错误

通过将剩余的逻辑分离进`run`函数而不是留在`main`中，就可以像列表 12-9 中的`Config::new`那样改进错误处理。不再通过通过`expect`允许程序 panic，`run`函数将会在出错时返回一个`Result<T, E>`。这让我们进一步以一种对用户友好的方式统一`main`中的错误处理。列表 12-12 展示了`run`签名和函数体中的变化：

<span class="filename">Filename: src/main.rs</span>

```rust,ignore
use std::error::Error;

// ...snip...

fn run(config: Config) -> Result<(), Box<Error>> {
    let mut f = File::open(config.filename)?;

    let mut contents = String::new();
    f.read_to_string(&mut contents)?;

    println!("With text:\n{}", contents);

    Ok(())
}
```

<span class="caption">Listing 12-12: Changing the `run` function to return
`Result`</span>

<!-- Will add ghosting and wingdings in libreoffice /Carol -->

这里做出了三个大的改变。第一，改变了`run`函数的返回值为`Result<(), Box<Error>>`。之前这个函数返回 unit 类型`()`，现在它仍然保持作为`Ok`时的返回值。

<!-- is just the `Box` bit the trait object, or the whole `Box<Error>`
syntax?-->
<!-- The whole `Box<Error>` /Carol -->

对于错误类型，使用了**trait 对象**`Box<Error>`（在开头使用了`use`语句将`std::error::Error`引入作用域）。第十七章会涉及 trait 对象。目前只需知道`Box<Error>`意味着函数会返回实现了`Error` trait 的类型，不过无需指定具体将会返回的值的类型。这提供了在不同的错误场景可能有不同类型的错误返回值的灵活性。

第二个改变是去掉了`expect`调用并替换为第九章讲到的`?`。不同于遇到错误就`panic!`，这会从函数中返回错误值并让调用者来处理它。

第三个修改是现在成功时这个函数会返回一个`Ok`值。因为`run`函数签名中声明成功类型返回值是`()`，这意味着需要将 unit 类型值包装进`Ok`值中。`Ok(())`一开始看起来有点奇怪，不过这样使用`()`是表明我们调用`run`只是为了它的副作用的惯用方式；它并没有返回什么有意义的值。

上述代码能够编译，不过会有一个警告：

```
warning: unused result which must be used, #[warn(unused_must_use)] on by default
  --> src/main.rs:39:5
   |
39 |     run(config);
   |     ^^^^^^^^^^^^
```

Rust 提示我们的代码忽略了`Result`值，它可能表明这里存在一个错误。虽然我们没有检查这里是否有一个错误，而编译器提醒我们这里应该有一些错误处理代码！现在就让我们修正他们。

#### 处理`main`中`run`返回的错误

我们将检查错误并使用与列表 12-10 中处理错误类似的技术来优雅的处理他们，不过有一些细微的不同：

<span class="filename">Filename: src/main.rs</span>

```rust,ignore
fn main() {
    // ...snip...

    println!("Searching for {}", config.query);
    println!("In file {}", config.filename);

    if let Err(e) = run(config) {
        println!("Application error: {}", e);

        process::exit(1);
    }
}
```

<!-- Will add ghosting and wingdings in libreoffice /Carol -->

我们使用`if let`来检查`run`是否返回一个`Err`值，不同于`unwrap_or_else`，并在出错时调用`process::exit(1)`。`run`并不返回像`Config::new`返回的`Config`实例那样需要`unwrap`的值。因为`run`在成功时返回`()`，而我们只关心发现一个错误，所以并不需要`unwrap_or_else`来返回未封装的值，因为它只会是`()`。

不过两个例子中`if let`和`unwrap_or_else`的函数体都一样：打印出错误并退出。

### 将代码拆分到库 crate

现在项目看起来好多了！现在我们将要拆分 *src/main.rs* 并将一些代码放入 *src/lib.rs*，这样就能测试他们并拥有一个小的`main`函数。

让我们将如下代码片段从 *src/main.rs* 移动到新文件 *src/lib.rs* 中：

- `run`函数定义
- 相关的`use`语句
- `Config`的定义
- `Config::new`函数定义

现在 *src/lib.rs* 的内容应该看起来像列表 12-13：

<span class="filename">Filename: src/lib.rs</span>

```rust,ignore
use std::error::Error;
use std::fs::File;
use std::io::prelude::*;

pub struct Config {
    pub query: String,
    pub filename: String,
}

impl Config {
    pub fn new(args: &[String]) -> Result<Config, &'static str> {
        if args.len() < 3 {
            return Err("not enough arguments");
        }

        let query = args[1].clone();
        let filename = args[2].clone();

        Ok(Config {
            query: query,
            filename: filename,
        })
    }
}

pub fn run(config: Config) -> Result<(), Box<Error>>{
    let mut f = File::open(config.filename)?;

    let mut contents = String::new();
    f.read_to_string(&mut contents)?;

    println!("With text:\n{}", contents);

    Ok(())
}
```

<span class="caption">Listing 12-13: Moving `Config` and `run` into
*src/lib.rs*</span>

<!-- Will add ghosting and wingdings in libreoffice /Carol -->

这里使用了公有的`pub`：在`Config`、其字段和其`new`方法，以及`run`函数上。现在我们有了一个拥有可以测试的公有 API 的库 crate 了。

#### 从二进制 crate 中调用库 crate

现在需要在 *src/main.rs* 中使用`extern crate greprs`将移动到 *src/lib.rs* 的代码引入二进制 crate 的作用域。接着我们将增加一个`use greprs::Config`行将`Config`类型引入作用域，并使用库 crate 的名称作为`run`函数的前缀，如列表 12-14 所示：

<span class="filename">Filename: src/main.rs</span>

```rust,ignore
extern crate greprs;

use std::env;
use std::process;

use greprs::Config;

fn main() {
    let args: Vec<String> = env::args().collect();

    let config = Config::new(&args).unwrap_or_else(|err| {
        println!("Problem parsing arguments: {}", err);
        process::exit(1);
    });

    println!("Searching for {}", config.query);
    println!("In file {}", config.filename);

    if let Err(e) = greprs::run(config) {
        println!("Application error: {}", e);

        process::exit(1);
    }
}
```

<span class="caption">Listing 12-14: Bringing the `greprs` crate into the scope
of *src/main.rs*</span>

<!-- Will add ghosting and wingdings in libreoffice /Carol -->



通过这些重构，所有功能应该抖联系在一起并可以运行了。运行`cargo run`来确保一切都正确的衔接在一起。

<!-- any tips for if they do find something is broken, main places to check? Or
just "diff your file against the XXX file in the book's resources to check
where it went wrong"? -->
<!-- We think general troubleshooting tips should be something we cover in
Chapter 1; the tips should apply to any example in the book /Carol -->

哇哦！这可有很多的工作，不过我们为将来成功打下了基础。现在处理错误将更容易，同时代码也更模块化。从现在开始几乎所有的工作都将在 *src/lib.rs* 中进行。

让我们利用这些新创建的模块的优势来进行一些在旧代码中难以展开的工作，他们在新代码中却很简单：编写测试！