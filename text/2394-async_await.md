- Feature Name: async_await
- Start Date: 2018-03-30
- RFC PR: [rust-lang/rfcs#2394](https://github.com/rust-lang/rfcs/pull/2394)
- Rust Issues: 
  - [rust-lang/rust#50547](https://github.com/rust-lang/rust/issues/50547)
  - [rust-lang/rust#62290](https://github.com/rust-lang/rust/issues/62290) - #!feature(async_closure)
- Translators:[[@FizzyElt](https://github.com/FizzyElt)]
- Commit: [The commit link this page based on](https://github.com/rust-lang/rfcs/blob/6041dd9ff2ed97eff4e054117cc1581b8c45d8c1/text/2394-async_await.md)
- Updated 2022-09-21

# 概要
[概要]: #summary

新增 async 和 await 語法，使撰寫程式碼操作 futures 更符合人因工程學。

這有一個[配套的 RFC](2592-futures.md)，用於向 libstd 和 libcore 新增一個小型 futures API。
# 動機
[動機]: #motivation

高效能網路服務經常使用非同步 IO，而不是阻塞 IO，因為在處理許多並行連接時更容易獲得最佳效能。 Rust 已經在網路服務領域得到了一些採用，我們希望透過使 Rust 中撰寫非同步網路服務更符合人因工程學，繼續支援這些使用者 - 並支援其他使用者採用。

Rust 中非同步 IO 的發展經歷了多個階段。在 1.0 之前，我們嘗試在語言中內嵌綠色執行緒（green-threading） runtime。然而，事實證明這太自以為是了——因為它影響了每個用 Rust 撰寫的程式——並且在 1.0 之前不久就被刪除了。在 1.0 之後，非同步 IO 最初專注於 mio 函式庫，它為 Linux、Mac OS 和 Windows 的非同步 IO 基元（primitive）提供了一個跨平台抽象。 2016 年年中，future crate 的引入產生了重大影響，它為非同步操作提供了一個方便、共享的抽象。 tokio 函式庫提供了一個基於 mio 的事件循環，可以執行使用 future 介面實作的程式碼。

在獲得基於 future 的生態系統的經驗和使用者的回饋後，我們發現了某些人因工程學挑戰。使用需要在等待點（await point）之間共享的狀態是非常不符合人因工程學的——需要 Arc 或 join chaining —— 雖然組合器通常比手動撰寫 future 更符合人因工程學，但它們仍然經常導致混亂的嵌套和 chained callbacks。

幸運的是，Future 抽象非常適合與一種語法糖一起使用，該語法糖在許多具有非同步 IO 的語言中變得很常見 - async 和 await 關鍵字。簡而言之，非同步函式返回一個 future，而不是在呼叫它時立即執行。在函式內部，可以使用 await 表達式等待其他 future，這使它們在輪詢 future 時讓出控制權。從使用者的角度來看，他們可以像使用同步程式碼一樣使用 async/await，並且只需要注釋他們的函式和調用。

Async/await 和 futures 通常是非同步和並行的強大抽象，並且可能在非同步 IO 空間之外有其他應用。我們今天遇到的案例通常與非同步 IO 相關，但透過引入一等公民語法和 libstd 的支援，我們相信更多的 async 和 await 案例也會蓬勃發展，而這些案例不直接與非同步 IO 相關。

# 教學式解說
[教學式解說]: #guide-level-explanation

## 非同步函式

函式可以添加 `async` 關鍵字，使它們成為「非同步函式」：

```rust
async fn function(argument: &str) -> usize {
     // ...
}
```

非同步函式的工作方式與普通函式不同。呼叫非同步函式時，它不會立即進入主體。相反，它執行實作 `Future` 特徵的匿名型別。當輪詢該 future 時，該函式被執行到它內部的下一個 `await` 或返回點（請參閱接下來 await 語法部分）。

非同步函式是延遲計算的一種 - 在您開始輪詢函式返回的 future 之前，函式本體中沒有任何內容被實際執行。例如：

```rust
async fn print_async() {
     println!("Hello from print_async")
}

fn main() {
     let future = print_async();
     println!("Hello from main");
     futures::executor::block_on(future);
}
```

這將在印出 `"Hello from main"` 之前印出 `"Hello from print_async"`。

`async fn foo(args..) -> T` 是 `fn(args..) -> impl Future<Output = T>` 型別的函式。返回型別是編譯器產生的匿名型別。

### `async ||` closures

除了函式，非同步也可以應用於 closure。與非同步函式一樣，非同步 closure 的返回行別為 `impl Future<Output = T>`，而不是 `T`。當您呼叫該 closure 時，它會立即返回一個 future 而不執行任何內容（就像非同步函式一樣）。

```rust
fn main() {
    let closure = async || {
         println!("Hello from async closure.");
    };
    println!("Hello from main");
    let future = closure();
    println!("Hello from main again");
    futures::block_on(future);
}
```

這將在印出 `"Hello from async closure."` 之前印出兩個 `"Hello from main"`。

`async` closure 可以用 `move` 來捕獲它們包覆在 closure 內的變數的所有權。

## `async` 區塊

您可以使用 `async` 區塊直接將 future 創建為表達式：

```rust
let my_future = async {
    println!("Hello from an async block");
};
```

這種形式幾乎等同於立即呼叫的 `async` closure。即是：

```rust
async { /* body */ }

// is equivalent to

(async || { /* body */ })()
```

除了像 `return`、`break` 和 `continue` 這樣的控制流結構不允許在 `body` 中使用（除非它們出現在一個新的控制流上下文中，比如 closure 或 loop）。 `?`-operator 和 early return 如何在非同步區塊中工作尚未確定（請參閱未解決的問題）。

與 `async` closure 一樣，`async` 區塊可以添加 `move`，以捕獲區塊內所包覆的變數的所有權。

## 編譯器內嵌的 `await!`

一個名為 `await!` 的內嵌函式被添加到編譯器。`await!` 可用於「暫停」future 的計算，將控制權交還給呼叫者。`await!` 接受任何實作 `IntoFuture` 的表達式，並計算為此 future 所傳入的泛型型別（如下面範例的 `Output`）之值。

```rust
// future: impl Future<Output = usize>
let n = await!(future);
```

await 的擴展在它接收到的 future 上重複呼叫 `poll`，當它返回 `Poll::Pending` 時產生對函式的控制，並最終在它返回 `Poll::Ready` 時獲取該項目值。

`await!` 只能在非同步函式、closure 或區塊內使用。在該上下文之外使用它是一個錯誤。

（`await!` 是一個內嵌編譯器，為以後確定其確切語法留出空間。可在未解決的問題部分中查看更多訊息。）

# 技術文件式解說
[技術文件式解說]: #reference-level-explanation

## 關鍵字

`async` 和 `await` 在 2018 版本中都成為關鍵字。

## `async` 函式、closure、區塊的回傳型別

非同步函式的回傳型別是編譯器生成的唯一匿名型別，類似於 closure 的型別。你可以把這種型別想像成一個枚舉，函式的每個「yield point」都有一個變體 —— 它的開頭、await 表達式和每個回傳。每個變體都儲存需要儲存的狀態，以便從該 yield point 恢復控制。

呼叫函式時，此匿名型別以其初始狀態返回，其中包含此函式的所有引數。

### 特徵綁定

匿名回傳型別實作 `Future`，`Item` 為它的回傳型別。輪詢它會推進函數的狀態，當它到達 `await` 時返回 `Pending` ，當它到達 `return` 時返回 `Ready` 。任何在它已經回傳 `Ready` 一次後對其嘗試進行輪詢都會造成恐慌。

匿名回傳型別對 `Unpin` 特徵有一個相反的實作 - 即 `impl !Unpin`。這是因為未來可能有內部引用，這意味著它永遠不需要被移動。

## 匿名 future 的生命週期捕捉

該函式的所有輸入生命週期都在非同步函式回傳的 future 捕捉，因為它將函式的所有引數儲存在其初始狀態（可能還有以後的狀態）。也就是說，給定這樣的函數：

```rust
async fn foo(arg: &str) -> usize { ... }
```

它具有與此等效的類型簽名：

```rust
fn foo<'a>(arg: &'a str) -> impl Future<Output = usize> + 'a { ... }
```

這與 `impl Trait` 的預設值不同，它不捕獲生命週期。這就是為什麼回傳類型是 `T` 而不是 `impl Future<Output = T>` 的一個重要部分。

### 「初始化」模式

有時會出現的一種模式是 future 有一個「初始化」步驟，應該在其創建期間執行。這在處理資料轉換和臨時借用時很有用。因為 async 函式在您輪詢它之前不會開始計算，並且它會捕獲其引數的生命週期，因此這種模式不能直接用 `async fn` 表示。

一種選擇是撰寫一個函式，該函式使用立即計算的 closure 返回 `impl Future`：

```rust
// only arg1's lifetime is captured in the returned future
fn foo<'a>(arg1: &'a str, arg2: &str) -> impl Future<Output = usize> + 'a {
    // do some initialization using arg2

    // closure which is evaluated immediately
    async move {
         // asynchronous portion of the function
    }
}
```

## await 的擴展

內嵌的 `await!` 大致上是這樣擴展的：

```rust
let mut future = IntoFuture::into_future($expression);
let mut pin = unsafe { Pin::new_unchecked(&mut future) };
loop {
    match Future::poll(Pin::borrow(&mut pin), &mut ctx) {
          Poll::Ready(item) => break item,
          Poll::Pending     => yield,
    }
}
```

這不是字面上的擴展，因為 `yield` 概念不能用 `async` 函式中的表面語法來表達。這就是為什麼 `await!` 是實作在編譯器內部，而不是實際的巨集。

## `async` 和 `move` 的順序

非同步 closure 和區塊可以用 `move` 註釋來捕獲它們包覆的變數的所有權。關鍵字的順序固定為 `async move`。只允許一種排序可以避免對其是否有重要意義的混淆。

```rust
async move {
    // body
}
```

# 缺點
[缺點]: #drawbacks

Adding async & await syntax to Rust is a major change to the language - easily
one of the most significant additions since 1.0. Though we have started with
the smallest beachhead of features, in the long term the set of features it
implies will grow as well (see the unresolved questions section). Such a
significant addition mustn't be taken lightly, and only with strong motivation.

We believe that an ergonomic asynchronous IO solution is essential to Rust's
success as a language for writing high performance network services, one of our
goals for 2018. Async & await syntax based on the Future trait is the most
expedient & low risk path to achieving that goal in the near future.

This RFC, along with its companion lib RFC, makes a much firmer commitment to
futures & async/await than we have previously as a project. If we decide to
reverse course after stabilizing these features, it will be quite costly.
Adding an alternative mechanism for asynchronous programming would be more
costly because this exists. However, given our experience with futures, we are
confident that this is the correct path forward.

There are drawbacks to several of the smaller decisions we have made as well.
There is a trade off between using the "inner" return type and the "outer"
return type, for example. We could have a different evaluation model for async
functions in which they are evaluated immediately up to the first await point.
The decisions we made on each of these questions are justified in the
appropriate section of the RFC.

# 原理和替代方案
[alternatives]: #alternatives

This section contains alternative design decisions which this RFC rejects (as
opposed to those it merely postpones).

## The return type (`T` instead of `impl Future<Output = T>`)

The return type of an asynchronous function is a sort of complicated question.
There are two different perspectives on the return type of an async fn: the
"interior" return type - the type that you return with the `return` keyword,
and the "exterior" return type - the type that the function returns when you
call it.

Most statically typed languages with async fns display the "outer" return type
in the function signature. This RFC proposes instead to display the "inner"
return type in the function signature. This has both advantages and
disadvantages.

### The lifetime elision problem

As alluded to previously, the returned future captures all input lifetimes. By
default, `impl Trait` does not capture any lifetimes. To accurately reflect the
outer return type, it would become necessary to eliminate lifetime elision:

```rust
async fn foo<'ret, 'a: 'ret, 'b: 'ret>(x: &'a i32, y: &'b i32) -> impl Future<Output = i32> + 'ret {
     *x + *y
}
```

This would be very unergonomic and make async both much less pleasant to use
and much less easy to learn. This issue weighs heavily in the decision to
prefer returning the interior type.

We could have it return `impl Future` but have lifetime capture work
differently for the return type of `async fn` than other functions; this seems
worse than showing the interior type.

### Polymorphic return (a non-factor for us)

According to the C# developers, one of the major factors in returning `Task<T>`
(their "outer type") was that they wanted to have async functions which could
return types other than `Task`. We do not have a compelling use case for this:

1. In the 0.2 branch of futures, there is a distinction between `Future` and
   `StableFuture`. However, this distinction is artificial and only because
   object-safe custom self-types are not available on stable yet.
2. The current `#[async]` macro has a `(boxed)` variant. We would prefer to
   have async functions always be unboxed and only box them explicitly at the
   call site. The motivation for the attribute variant was to support async
   methods in object-safe traits. This is a special case of supporting `impl
   Trait` in object-safe traits (probably by boxing the return type in the
   object case), a feature we want separately from async fn.
3. It has been proposed that we support `async fn` which return streams.
   However, this mean that the semantics of the internal function would differ
   significantly between those which return futures and streams. As discussed
   in the unresolved questions section, a solution based on generators and
   async generators seems more promising.

For these reasons, we don't think there's a strong argument from polymorphism
to return the outer type.

### Learnability / documentation trade off

There are arguments from learnability in favor of both the outer and inner
return type. One of the most compelling arguments in favor of the outer return
type is documentation: when you read automatically generated API docs, you will
definitely see what you get as the caller. In contrast, it can be easier to
understand how to write an async function using the inner return type, because
of the correspondence between the return type and the type of the expressions
you `return`.

Rustdoc can handle async functions using the inner return type in a couple of
ways to make them easier to understand. At minimum we should make sure to
include the `async` annotation in the documentation, so that users who
understand async notation know that the function will return a future. We can
also perform other transformations, possibly optionally, to display the outer
signature of the function. Exactly how to handle API documentation for async
functions is left as an unresolved question.

## Built-in syntax instead of using macros in generators

Another alternative is to focus on stabilizing procedural macros and
generators, rather than introducing built-in syntax for async functions. An
async function can be modeled as a generator which yields `()`.

In the long run, we believe we will want dedicated syntax for async functions,
because it is more ergonomic & the use case is compelling and significant
enough to justify it (similar to - for example - having built in for loops and
if statements rather than having macros which compile to loops and match
statements). Given that, the only question is whether or not we could have a
more expedited stability by using generators for the time being than by
introducing async functions now.

It seems unlikely that using macros which expand to generators will result in a
faster stabilization. Generators can express a wider range of possibilities,
and have a wider range of open questions - both syntactic and semantic. This
does not even address the open questions of stabilizing more procedural macros.
For this reason, we believe it is more expedient to stabilize the minimal
built-in async/await functionality than to attempt to stabilize generators and
proc macros.

## `async` based on generators alone

Another alternative design would be to have async functions *be* the syntax for
creating generators. In this design, we would write a generator like this:

```rust
async fn foo(arg: Arg) -> Return yield Yield
```

Both return and yield would be optional, default to `()`. An async fn that
yields `()` would implement `Future`, using a blanket impl. An async fn that
returns `()` would implement `Iterator`.

The problem with this approach is that does not ergonomically handle `Stream`s,
which need to yield `Poll<Option<T>>`. It's unclear how `await` inside of an
async fn yielding something other than `()` (which would include streams) would
work.  For this reason, the "matrix" approach in which we have independent
syntax for generator functions, async functions, and async generator functions,
seems like a more promising approach.

## "Hot async functions"

As proposed by this RFC, all async functions return immediately, without
evaluating their bodies at all. As discussed above, this is not convenient for
use cases in which you have an immediate "initialization" step - those use
cases need to use a terminal async block, for example.

An alternative would be to have async functions immediately evaluate up until
their first `await`, preserving their state until then. The implementation of
this would be quite complicated - they would need to have an additional yield
point within the `await`, prior to polling the future being awaited,
conditional on whether or not the await is the first await in the body of the
future.

A fundamental difference between Rust's futures and those from other languages
is that Rust's futures do not do anything unless polled. The whole system is
built around this: for example, cancellation is dropping the future for
precisely this reason. In contrast, in other languages, calling an async fn
spins up a future that starts executing immediately. This difference carries
over to `async fn` and `async` blocks as well, where it's vital that the
resulting future be *actively polled* to make progress. Allowing for partial,
eager execution is likely to lead to significant confusion and bugs.

This is also complicated from a user perspective - when a portion of the body
is evaluated depends on whether or not it appears before all `await`
statements (which could possibly be macro generated). The use of a terminal
async block provide a clearer mechanism for distinguishing between the
immediately evaluated and asynchronously evaluated portions of a future with an
initialization step.

## Using async/await instead of alternative asynchronicity systems

A final - and extreme - alternative would be to abandon futures and async/await
as the mechanism for async/await in Rust and to adopt a different paradigm.
Among those suggested are a generalized effects system, monads & do notation,
green-threading, and stack-full coroutines.

While it is hypothetically plausible that some generalization beyond
async/await could be supported by Rust, there has not enough research in this
area to support it in the near-term. Given our goals for 2018 - which emphasize
shipping - async/await syntax (a concept available widely in many languages
which interacts well with our existing async IO libraries) is the most logical
thing to implement at this stage in Rust's evolution.

## Async blocks vs async closures

As noted in the main text, `async` blocks and `async` closures are closely
related, and are roughly inter-expressible:

```rust
// almost equivalent
async { ... }
(async || { ... })()

// almost equivalent
async |..| { ... }
|..| async { ... }
```

We could consider having only one of the two constructs. However:

- There's a strong reason to have `async ||` for consistency with `async fn`;
  such closures are often useful for higher-order constructs like constructing a
  service.

- There's a strong reason to have `async` blocks: The initialization pattern
  mentioned in the RFC text, and the fact that it provides a more
  direct/primitive way of constructing futures.

The RFC proposes to include both constructs up front, since it seems inevitable
that we will want both of them, but we can always reconsider this question
before stabilization.

# 現有技術
[現有技術]: #prior-art

There is a lot of precedence from other languages for async/await syntax as a
way of handling asynchronous operation - notable examples include C#,
JavaScript, and Python.

There are three paradigms for asynchronous programming which are dominant
today:

- Async and await notation.
- An implicit concurrent runtime, often called "green-threading," such as
  communicating sequential processes (e.g. Go) or an actor model (e.g. Erlang).
- Monadic transformations on lazily evaluated code, such as do notation (e.g.
  Haskell).

Async/await is the most compelling model for Rust because it interacts
favorably with ownership and borrowing (unlike systems based on monads) and it
enables us to have an entirely library-based asynchronicity model (unlike
green-threading).

One way in which our handling of async/await differs from most other statically
typed languages (such as C#) is that we have chosen to show the "inner" return
type, rather than the outer return type. As discussed in the alternatives
section, Rust's specific context (lifetime elision, the lack of a need for
return type polymorphism here) make this deviation well-motivated.

# 未解決的問題
[未解決的問題]: #unresolved-questions

This section contains design extensions which have been postponed & not
included in this initial RFC.

## Final syntax for the `await` expression

Though this RFC proposes that `await` be a built-in macro, we'd prefer that
some day it be a normal control flow construct. The unresolved question about
this is how to handle its precedence & whether or not to require delimiters of
some kind.

In particular, `await` has an interesting interaction with `?`. It is very
common to have a future which will evaluate to a `Result`, which the user will
then want to apply `?` to. This implies that await should have a tighter
precedence than `?`, so that the pattern will work how users wish it to.
However, because it introduces a space, it doesn't look like this is the
precedence you would get:

```
await future?
```

There are a couple of possible solutions:

1. Require delimiters of some kind, maybe braces or parens or either, so that
   it will look more like how you expect - `await { future }?` - this is rather
   noisy.
2. Define the precedence as the obvious, if inconvenient precedence, requiring
   users to write `(await future)?` - this seems very surprising for users.
3. Define the precedence as the inconvenient precedence - this seems equally
   surprising as the other precedence.
4. Introduce a special syntax to handle the multiple applications, such as
   `await? future` - this seems very unusual in its own way.

This is left as an unresolved question to find another solution or decide which
of these is least bad.

## `for await` and processing streams

Another extension left out of the RFC for now is the ability to process streams
using a for loop. One could imagine a construct like `for await`, which takes
an `IntoStream` instead of an `IntoIterator`:

```rust
for await value in stream {
    println!("{}", value);
}
```

This is left out of the initial RFC to avoid having to stabilize a definition
of `Stream` in the standard library (to keep the companion RFC to this one as
small as possible).

## Generators and Streams

In the future, we may also want to be able to define async functions that
evaluate to streams, rather than evaluating to futures. We propose to handle
this use case by way of generators. Generators can evaluate to a kind of
iterator, while async generators can evaluate to a kind of stream.

For example (using syntax which could change);

```rust
// Returns an iterator of i32
fn foo(mut x: i32) yield i32 {
     while x > 0 {
          yield x;
          x -= 2;
     }
}

// Returns a stream of i32
async fn foo(io: &AsyncRead) yield i32 {
    async for line in io.lines() {
        yield line.unwrap().parse().unwrap();
    }
}
```

## Async functions which implement `Unpin`

As proposed in this RFC, all async functions do not implement `Unpin`, making
it unsafe to move them out of a `Pin`. This allows them to contain references
across yield points.

We could also, with an annotation, typecheck an async function to confirm that it
does not contain any references across yield points, allowing it to implement
`Unpin`. The annotation to enable this is left unspecified for the time being.

## `?`-operator and control-flow constructs in async blocks

This RFC does not propose how the `?`-operator and control-flow constructs like
`return`, `break` and `continue` should work inside async blocks.

It was discussed that async blocks should act as a boundary for the
`?`-operator. This would make them suitable for fallible IO:

```rust
let reader: AsyncRead = ...;
async {
    let foo = await!(reader.read_to_end())?;
    Ok(foo.parse().unwrap_or(0))
}: impl Future<Output = io::Result<u32>>
```

Also, it was discussed to allow the use of `break` to return early from
an async block:

```rust
async {
    if true { break "foo" }
}
```

The use of the `break` keyword instead of `return` could be beneficial to
indicate that it applies to the async block and not its surrounding function. On
the other hand this would introduce a difference to closures and async closures
which make use the `return` keyword.
