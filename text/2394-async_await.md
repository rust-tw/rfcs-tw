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

另有一個[配套 RFC](2592-futures.md)，用於向 libstd 和 libcore 新增一個小型 futures API。
# 動機
[動機]: #motivation

高效能網路服務經常使用非同步 IO，而不是阻塞式 IO，這樣在處理許多並行連線時更容易獲得更好的效能表現。 Rust 已經在網路服務領域得到了一些採用，我們希望透過使 Rust 中撰寫非同步網路服務更符合人因工程學，繼續支援這些使用者 - 並支援其他使用者採用。

Rust 中非同步 IO 的發展經歷了多個階段。在 1.0 之前，我們嘗試在語言中內嵌綠色執行緒（green-threading） runtime。然而，事實證明這太過武斷了——因為它影響了每個用 Rust 撰寫的程式——並且在 1.0 之前不久就被刪除了。在 1.0 之後，非同步 IO 最初的重點是 mio 函式庫，它為 Linux、Mac OS 和 Windows 的非同步 IO 基元（primitive）提供了一個跨平台抽象。 2016 年年中，future crate 的引入產生了重大影響，它為非同步操作提供了一個方便且可共享的抽象。 tokio 函式庫提供了一個基於 mio 的事件循環，可以執行使用 future 介面實作的程式碼。

在獲得基於 future 的生態系統的經驗和使用者的回饋後，我們發現了某些人因工程學挑戰。使用需要在等待點（await point）之間共享的狀態，是非常不符合人因工程學的（需要 Arc 或 join chaining）雖然組合器通常比手動撰寫 future 更符合人因工程學，但它們仍然經常導致混亂的嵌套和 chained callbacks。

幸運的是，Future 抽象非常適合與一種語法糖一起使用——這個語法糖在許多具有非同步 IO 的語言中愈來愈常見：async 和 await 關鍵字。簡而言之，非同步函式回傳一個 future，而不是在呼叫時立即執行。在函式內部，可以使用 await 表達式等待其他 future，這使它們在輪詢 future 時讓出控制權。從使用者的角度來看，他們可以像在使用同步程式碼一樣使用 async/await，並且只需要標注其函式和調用。

Async/await 和 futures 通常是非同步和並行的強大抽象，並且可能可以應用在非同步 I/O 空間以外的地方。我們今天遇到的案例通常與非同步 IO 相關，但透過引入一等公民語法和 libstd 的支援，我們相信更多不直接與非同步 IO 相關的 async 和 await 案例也會蓬勃發展。

# 教學式解說
[教學式解說]: #guide-level-explanation

## 非同步函式

對函式加上 `async` 關鍵字，使它們成為「非同步函式」：

```rust
async fn function(argument: &str) -> usize {
     // ...
}
```

非同步函式的工作方式與普通函式不同。呼叫非同步函式時，它不會立即進入主體。相反，它執行實作 `Future` 特徵的匿名型別。當輪詢該 future 時，該函式被執行到它內部的下一個 `await` 或回傳點（請參閱接下來 await 語法部分）。

非同步函式是延遲計算的一種 - 在您開始輪詢函式回傳的 future 之前，函式本體中沒有任何內容被實際執行。例如：

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

`async fn foo(args..) -> T` 是 `fn(args..) -> impl Future<Output = T>` 型別的函式。回傳型別是編譯器產生的匿名型別。

### `async ||` closures

除了函式，非同步也可以應用在 closure 上面。與非同步函式一樣，非同步 closure 的回傳型別為 `impl Future<Output = T>`，而不是 `T`。當您呼叫該 closure 時，它會立即回傳一個 future，且不會執行任何程式碼（就像非同步函式一樣）。

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

`async` closure 可以用 `move` 來捕捉它們包覆在 closure 內的變數的所有權。

## `async` 區塊

您可以使用 `async` 區塊直接將 future 建立為表達式：

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

除了像 `return`、`break` 和 `continue` 這樣的控制流程結構不允許在 `body` 中使用（除非它們出現在一個新的控制流上下文中，比如 closure 或 loop）。 尚未確定 `?` 運算子和提早回傳（early return）在非同步區塊運作的方式（請參閱未解決的問題）。

與 `async` closure 一樣，`async` 區塊可以加入 `move`，以捕捉區塊內所包覆的變數的所有權。

## 編譯器內嵌的 `await!`

編譯器加入了一個名為 `await!` 的內建函式。`await!` 可用於「暫停」future 的計算，將控制權交還給呼叫者。`await!` 接受任何實作 `IntoFuture` 的表達式，並計算為此 future 所傳入的泛型型別（如下面範例的 `Output`）之值。

```rust
// future: impl Future<Output = usize>
let n = await!(future);
```

await 展開的程式碼，會重複在接收到的 future 上呼叫 `poll`：`poll` 回傳 `Poll::Pending` 時讓出 (yield) 函式的控制權，並在最終回傳 `Poll::Ready` 時取得項目的值。

`await!` 只能在非同步函式、closure 或區塊內使用，除此之外使用它都是錯誤的。

（`await!` 是編譯器的內建函式，為以後確定其確切語法保留彈性空間。詳細資訊請參閱〈未解決的問題〉部分。）

# 技術文件式解說
[技術文件式解說]: #reference-level-explanation

## 關鍵字

`async` 和 `await` 在 2018 版本中都成為關鍵字。

## `async` 函式、closure、區塊的回傳型別

非同步函式的回傳型別是編譯器生成的唯一匿名型別，和 closure 的型別類似。你可以把這種型別想像成一個枚舉，函式的每個「yield point」都是一個變體——開頭、await 表達式和每一次的回傳。每個變體都會儲存需要保存的狀態，以便從該 yield point 恢復控制。

呼叫函式時，此匿名型別以其初始狀態回傳，其中包含此函式的所有引數。

### 特徵綁定

匿名回傳型別實作 `Future`，`Item` 為它的回傳型別。輪詢它會推進函數的狀態，當它在 `await` 狀態時會返回 `Pending`；當它在 `return` 狀態時則返回 `Ready`。任何在它已經回傳 `Ready` 一次後對其嘗試進行輪詢都將造成恐慌。

匿名回傳型別對 `Unpin` 特徵有一個相反的實作，即 `impl !Unpin`。這是因為 future 可能有內部引用，這意味著它永遠不需要被移動。

## 匿名 future 的生命週期捕捉

該函式的所有輸入生命週期都在非同步函式回傳的 future 捕捉，因為它將函式的所有引數儲存在其初始狀態（可能還有以後的狀態）。也就是說，給定這樣的函數：

```rust
async fn foo(arg: &str) -> usize { ... }
```

它具有與此等效的類型簽名：

```rust
fn foo<'a>(arg: &'a str) -> impl Future<Output = usize> + 'a { ... }
```

這與 `impl Trait` 的預設值不同，它不捕捉生命週期。這就是為什麼回傳類型是 `T` 而不是 `impl Future<Output = T>` 的一個重要部分。

### 「初始化」模式

有時會出現的一種模式是 future 有一個「初始化」步驟，應該在其建立期間被執行。這在處理資料轉換和臨時借用時很有用。因為 async 函式在您輪詢它之前不會開始計算，並且它會捕捉其引數的生命週期，因此這種模式不能直接用 `async fn` 表示。

其中一個解決辦法，是撰寫一個回傳 `impl Future` 的函式，而回傳值是會立即計算 (evaluate) 的 closure：

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

## await 展開後的程式碼

內嵌的  `await!` 展開結果大致如下：

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

這不是真正意義上的『展開』，因為 `yield` 概念不能用 `async` 函式中的表層語法來表達。這就是為什麼 `await!` 是一個內建編譯器函式，而不是實際的巨集。

## `async` 和 `move` 的順序

非同步 closure 和區塊可以用 `move` 註釋來捕捉它們包覆的變數的所有權。關鍵字的順序固定為 `async move`。只允許一種順序，可以避免語義上「是否有重要意義」的混淆。

```rust
async move {
    // body
}
```

# 缺點
[缺點]: #drawbacks

在 Rust 中新增 async 和 await 語法是對語言的重大更改 - 這是自 1.0 以來最重要的新增功能之一。雖然我們從最小的功能開始，但從長遠來看，它所隱含的功能集也會增長（請參閱未解決的問題部分）。對於這樣一個重要的新增功能，我們絕不能掉以輕心，只有在強烈的動機下才能進行。

我們相信，一個符合人因工程學的非同步 IO 解決方案對於 Rust 作為撰寫高效能網路服務的語言的成功至關重要，這是我們 2018 年的目標之一。基於 Future trait 的 async & await 語法是在不久的將來實現這一目標最便捷和低風險的途徑。

這個 RFC，連同其配套的 lib RFC，對 future 和 async/await 做出比我們過往的專案更堅定的承諾。如果我們在穩定這些特性之後決定反其道而行，那將付出相當大的代價。因為這個 RFC 的存在，增加非同步程式的替代機制的成本會更高。然而，有鑑於我們在 future 方面的經驗，我們相信這是正確的發展方向。

我們所做的幾個小決定也有缺點。例如，在使用「内部」回傳型別和「外部」回傳型別之間有一個權衡。我們可以為非同步函式建立一個不同的求值模型，即在第一個等待（await point）點之前立即對其進行求值。我們在這些問題上做出的决定在 RFC 的相應部分都有說明。

# 原理和替代方案
[alternatives]: #alternatives

本節包含了本 RFC 拒絕的替代性設計決定（相對於那些只是推遲的設計）。

## 回傳型別（使用 `T` 而不是 `impl Future<Output = T>`）

非同步函式的回傳型別是一個有點複雜的問題。對於非同步函式的回傳型別，有兩個不同的觀點：「内部」回傳型別 - 你用 `return` 關鍵字回傳的型別，以及「外部」回傳型別 - 當你呼叫函式時回傳的型別。

大多數帶有非同步函式的靜態型別語言在函式簽名中顯示「外部」回傳型別。本 RFC 建議在函式簽名中顯示「内部」回傳型別。這既有優點也有缺點。

### 生命週期消除問題

正如前面所提到的，回傳的 future 捕捉了所有傳入的生命週期。預設情況下，`impl Trait` 不捕捉任何生命週期。為了準確反應外部回傳型別，有必要消除生命週期的省略：

```rust
async fn foo<'ret, 'a: 'ret, 'b: 'ret>(x: &'a i32, y: &'b i32) -> impl Future<Output = i32> + 'ret {
     *x + *y
}
```

這將是非常不符合人因工程學的，並且使非同步的使用變得更不愉快，更不易於學習。這個問題在決定回傳內部型別時佔很大比重。

我們可以讓它回傳 `impl Future`，但對於 `async fn` 的回傳型別，生命週期捕捉的運作方式與其他函式不同，這似乎比顯示內部型別更糟糕。

### 多型的回傳（對我們來說並非是一個因素）

根據 C# 開發者的說法，回傳 `Task<T>`（他們的「外部型別」）的主要因素之一是，他們希望有可以回傳 `Task` 以外型別的非同步函式。我們對此沒有一個可以令人信服的實例。

1. 在 future 的 0.2 分支中，`Future` 和 `StableFuture` 之間是有區別的。然而，這種區分是人為的，單純是因為物件安全（object-safe）的自定義自型別（self-types）在穩定版本上還不能使用。
2. 目前的 `#[async]` 巨集有一個 `(boxed)` 變體。我們更傾向於讓非同步函式盡可能的不包裝，只在呼叫處明確包裝。屬性變體的動機是為了支援物件安全特徵中的非同步方法。這是在物件安全特徵中支援 `impl Trait` 的一個特例（可能是透過在物件情况下對回傳型別進行包裝），我們希望這個特性與非同步函式分開。
3. 有人建議我們支援回傳串流（stream）的 `async fn`。然而，這意味著内部函式的語意在回傳 future 和串流的函式之間會有顯著的不同。正如在未解决的問題部分所討論的，基於生成器和非同步生成器的解決方案似乎更有機會。

基於這些原因，我們認為從多型的角度來看，回傳外部型別的論點並不強烈。

### 可學習性/文件的權衡

從可學習性的角度來看，支援外部和内部回傳型別的論點都有。支援外部回傳型別最有說服力的論據之一是文件：當你閱讀自動產生的 API 文件時，你肯定會看到你作為呼叫者得到的東西。相較之下，由於回傳型別和你 `return` 的表達式的型別之間的對應關係，可以更容易理解如何使用内部回傳型別撰寫非同步函式。

Rustdoc 可以透過幾種方式處理使用内部回傳型別的非同步函式，使其更容易理解。我們至少應該確保在文件中包含 `async` 註解，這樣了解 async 符號的使用者就知道此函式將回傳一個 future。我們還可以進行其他轉換，可能是可選的，以顯示函式的外部簽名。如何確切處理非同步函式的 API 文件是一個尚未解决的問題。

## 內嵌語法，而不是在生成器中使用巨集

另一個選擇是專注於穩定程序性巨集（procedural macro）和生成器，而不是為非同步函式引入內嵌語法。一個非同步函式可以被建模為一個生成器，它將產生 `()`。

從長遠來看，我們相信我們會希望有專門的語法來處理非同步函式，因為它更符合人因工程學原理，而且使用情境也足夠令人信服和重要，可以證明這一點（類似於 - 例如 - 有内嵌的 for 迴圈和 if 判斷式，而不是有編譯成迴圈和配對判斷式的巨集）。鑑於此，唯一的問題是，我們是否可以透過暫時使用生成器來獲得比現在引入非同步函式更好的穩定性。

使用展開到生成器的巨集似乎不太可能使其更快的穩定。生成器可以表現更多的可能性，並且有更多的開放性問題 - 包括語法和語意。這甚至沒有解決穩定更多程序性巨集的開放問題。出於這個原因，我們認為穩定最小的内嵌 async/await 功能比試圖穩定生成器和 proc 巨集更有效益。

## 單純基於生成器的 `async`

另一種設計是將非同步函式作為建立生成器的語法。在這種設計中，我們可以寫一個這樣的生成器：

```rust
async fn foo(arg: Arg) -> Return yield Yield
```

return 和 yield 都是可選的，預設為 `()`。一個產生 `()` 的非同步函式將使用全面實作（blanket impl）來實作 `Future`。一個回傳 `()` 的非同步函式將實作 `Iterator`。

此方法的問題是，它不能從人因工程學的角度處理 `Stream`，Stream 需要產生 `Poll<Option<T>>`。目前還不清楚在一個產生 `()` 以外的東西（包括 Stream）的非同步函式裡的 `await` 如何運作。由於這個原因，「矩陣」方法，即我們對生成器函式、非同步函式和非同步生成器函式有獨立的語法，似乎是一個更可行的方法。

## "Hot async functions"

正如本 RFC 所建議的，所有的非同步函式都會立即回傳，根本不需要執行其主體。如上所述，這對於需要立即進行「初始化」步驟的情境來說並不方便 - 例如，這些情境需要使用一個終端非同步區塊。

另一種方法是讓非同步函式立即求值，直到它們的第一個 `await`，在那之前保留它們的狀態。這將是一個相當複雜的實現 - 它們需要在 `await` 中擁有一个額外的 yield point，在輪詢被 await 的 future 之前，條件是 await 是否是 future 主體中的第一個 await。

Rust 的 future 與其它語言的 future 的一個根本區別是，Rust 的 future 除非被輪詢，否則不會做任何事情。整個系统都是圍繞這一點建立的：例如，取消正是因為這個原因而捨棄了 future。相反，在其它語言中，呼叫一個非同步函式會產生一個立即開始執行的 future。這種差異也延續到了 `async fn` 和 `async` 區塊中，其中至關重要的是，產生的 future 要**主動輪詢**以取得進展。允許部分、急迫的執行很可能會引發嚴重的混亂和錯誤。

從使用者的角度來看，這也很複雜 - 主體的一部分何時被執行取決於它是否出現在所有 `await` 語句（可能是巨集生成的）之前。使用终端 async 區塊提供了一個更清晰的機制來區分帶有初始化步驟的 future 中立即執行部分和非同步執行部分。

## 使用 async/await 而不是其他的非同步性系統

最後，一個極端的選擇是放棄 future 和 async/await 作為 Rust 中 async/await 的機制，而採用不同的典範。在這些建議中，有一個常見的效果系统，monad 和 do 語法（do notation）、綠色執行緒和滿堆疊（stack-full）的協程。

假設性上來說，Rust 可以透過 async/await 語法來達成一些泛化（generalization），但在這個領域的研究還不足以在短期内支援它。考慮到我們 2018 年的目標 - 強調 - async/await 語法（一個在許多語言中廣泛存在的概念，與我們現有的 async IO 函式庫運作良好）是在 Rust 發展的這個階段最合理的實作。

## 非同步區塊與非同步 closure

正如文中所指出的，非同步區塊和非同步 closure 是密切相關的，而且大致上是可以相互表達的：

```rust
// almost equivalent
async { ... }
(async || { ... })()

// almost equivalent
async |..| { ... }
|..| async { ... }
```

我們可以考慮只採用兩個結構中的其中一個。然而：

- 為了與 `async fn` 保持一致，我們有充分的理由使用 `async ||`；這樣的 closure 通常對建構一個服務這樣的高階構造很有用。

- 有一個強而有力的理由讓我們採用非同步區塊。RFC 文件中提到的初始化模式，以及事實上它提供了一種更直接、更原始的方式建立 future。

RFC 提議在前面就包含這兩個構造，因為我們似乎不可避免地需要這兩者，但我們總是可以在穩定之前重新思考這個問題。

# 現有技術
[現有技術]: #prior-art

在其他語言中，包含 C#、JavaScript 和 Python，有很多關於使用 async/await 語法作為處理非同步操作的一種方式的先例。

目前主流的非同步程式設計有以下三種範式：

- async 和 await 符號。
- 隱式並行的執行階段程式庫（implicit concurrent runtime），通常稱為「綠色執行緒」，例如通信順序行程（例如 Go）或參與者模型（例如 Erlang）。
- 延遲執行程式中的 Monadic 轉換，例如：Haskell 的 do 語法（do notation）。

async/await 是 Rust 最引人注目的模型，因為它與所有權和借用互動良好（不像基於 Monadic 的系统），而且它使我們能夠擁有一個完全基於函式庫的非同步模型（不像綠色執行緒）。

我們對 async/await 的處理不同於大多數其他靜態型別語言（例如 C#），我們選擇顯示「內部」回傳型別，而不是外部回傳型別。正如在替代方案部分中討論的那樣，在 Rust 定義的特定脈絡下（生命週期省略，不需要回傳型別多型），這種偏差的動機充分。

# 未解決的問題
[未解決的問題]: #unresolved-questions

本節包含已推延且未包含在此初始 RFC 中設計的延伸。

## `await` 表達式的最終語法

儘管此 RFC 建議 `await` 是一個內建的巨集，但我們希望有一天它成為一個正常的控制流結構。如何處理它的運算子優先順序，以及是否需要某種分隔符號則尚待解決。

特別是， `await` 與 `?` 有一個有趣的互動。很常見的情況是有一個 future，它將被執行為一個 `Result`，然後使用者會想把這個结果應用到 `?` 這意味著 await 應該比 `?` 有更高的優先順序，這樣該模式就能按照使用者的意願運作。然而，由於它引入了一個空格，看起來這並不是你要得到的優先順序：

```
await future?
```

以下有幾種可能的解決方案：

1. 需要某種的分隔符號，可能是大括號或括號或兩者之一，讓它看起來更符合期望的那樣 - await {future}？ - 這很煩躁。
2. 將優先順序定義為，如果優先順序不符使用者本意，需要使用者明確指出 `(await future)?` - 對使用者來說非常令人驚訝。
3. 將其定義為不方便的優先順序 —— 這似乎與其他優先順序一樣令人驚訝。
4. 引入一種特殊的語法來處理多個應用程式，例如 `await? future` - 這似乎是很不尋常的方式。

這個問題留給未來找尋另一種解決方案，或是從上述方案中選擇最不糟糕的一個。

## `for await` 和處理串流

RFC 目前遺漏的另一個延伸是使用 for 迴圈處理串流的能力。可以想像 `for await` 這樣的結構，它採用 `IntoStream` 而不是 `IntoIterator`：

```rust
for await value in stream {
    println!("{}", value);
}
```

這被排除在最初的 RFC 之外，以避免必須在標準庫中穩定 `Stream` 的定義（以使與此相關的 RFC 盡可能小）。

## 生成器和串流

將來，我們可能還希望能夠定義對串流求值非同步函式，而不是對 future 求值。我們建議透過生成器來處理這個案例。生成器可以轉換為一種迭代器，而非同步生成器可以轉換為一種串流。

例如（使用的語法可能會改變）；

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

## 實現 `Unpin` 的非同步函式

如本 RFC 中所提議，所有非同步函式均未實現 `Unpin`，因此將它們從 `Pin` 中移出是不安全的。這允許它們包含跨越 yield point 的引用。

我們還可以透過註釋對非同步函式進行型別檢查，以確認它不包含任何跨越 yield point 的引用，從而允許它實做 `Unpin`。可啟用此功能的註釋暫時未指定。
## 異步區塊中的 `?` 運算子和控制流構造

這個 RFC 沒有提出 `?` 運算子和控制流結構如 `return`、`break` 和 `continue` 應該如何在非同步區塊中工作。

不過有討論非同步區塊應該充當 `?` 運算子的邊界。讓它們適用於易出錯的 IO：

```rust
let reader: AsyncRead = ...;
async {
    let foo = await!(reader.read_to_end())?;
    Ok(foo.parse().unwrap_or(0))
}: impl Future<Output = io::Result<u32>>
```

此外，還討論了允許使用 break 從非同步區塊中提前回傳：

```rust
async {
    if true { break "foo" }
}
```

使用 `break` 關鍵字而不是 `return` 可能有助於表明它適用於非同步區塊而不是其周圍的函式。另一方面，這會給使用 `return` 關鍵字的 closure 和非同步 closure 帶來區別。
