- Feature Name: scoped_threads
- Start Date: 2019-02-26
- RFC PR: [rust-lang/rfcs#3151](https://github.com/rust-lang/rfcs/pull/3151)
- Rust Issue: [rust-lang/rust#93203](https://github.com/rust-lang/rust/issues/93203)
- Translators: [[@FizzyElt](https://github.com/FizzyElt)]
- Commit: []()
- Updated: 2022-07-05

# 概要
[概要]: #summary

在標準函式庫中新增作用域執行緒，其允許借用父線程的變數來產生線程。

範例:

```rust
let var = String::from("foo");

thread::scope(|s| {
    s.spawn(|_| println!("borrowed from thread #1: {}", var));
    s.spawn(|_| println!("borrowed from thread #2: {}", var));
});
```

# 動機
[動機]: #motivation

在 Rust 1.0 發布之前，我們有與作用域執行續相同作用的 [`thread::scoped()`](https://docs.rs/thread-scoped/1.0.2/thread_scoped/)，但後來發現一個健全性的問題，可能導致 use-after-frees，所以它已被移除。這一歷史事件稱為[洩密事件](http://cglab.ca/~abeinges/blah/everyone-poops/)。

幸運的是，舊的作用域執行緒可以透過依靠閉包而不是守護來確保生成的線程被自動加入。但我們對在 Rust 1.0 中加入的作用域執行緒並不放心，所以我們決定將其放在外部 crates 之中，並有可能在未來的某個時刻回到標準函式庫中。四年過去了，那個未來就是現在。

作用域執行緒在 [Crossbeam](https://docs.rs/crossbeam/0.7.1/crossbeam/thread/index.html) 中經過多年的經驗累積，我們的設計已經趨於成熟，足以被推廣到標準函式庫中。

更多內容請看[基本理由及替代方案](#rationale-and-alternatives)部分。

# 指導級解釋
[指導級解釋]: #guide-level-explanation

執行緒生成的 "hello world" 可能如下所示：

```rust
let greeting = String::from("Hello world!");

let handle = thread::spawn(move || {
    println!("thread #1 says: {}", greeting);
});

handle.join().unwrap();
```

現在讓我們嘗試生成兩個使用相同 greeting 的線程。不幸的是，我們必須克隆它，因為 [`thread::spawn()`](https://doc.rust-lang.org/std/thread/fn.spawn.html) 有 `F: 'static` 要求，這意味著執行緒不能借用局部變數：

```rust
let greeting = String::from("Hello world!");

let handle1 = thread::spawn({
    let greeting = greeting.clone();
    move || {
        println!("thread #1 says: {}", greeting);
    }
});

let handle2 = thread::spawn(move || {
    println!("thread #2 says: {}", greeting);
});

handle1.join().unwrap();
handle2.join().unwrap();
```

作用域執行緒來解決！透過打開一個新的 `thread::scope()` 區塊，我們可以向編譯器證明在這作用域內產生的所有執行緒也會在這個作用域內結束生命。

```rust
let greeting = String::from("Hello world!");

thread::scope(|s| {
    let handle1 = s.spawn(|_| {
        println!("thread #1 says: {}", greeting);
    });

    let handle2 = s.spawn(|_| {
        println!("thread #2 says: {}", greeting);
    });

    handle1.join().unwrap();
    handle2.join().unwrap();
});
```

這意味著可以毫無顧忌地借用作用域之外的變數！

現在我們不必再手動加入執行緒，因為所有未加入的執行緒將在作用域結束時自動加入：

```rust
let greeting = String::from("Hello world!");

thread::scope(|s| {
    s.spawn(|_| {
        println!("thread #1 says: {}", greeting);
    });

    s.spawn(|_| {
        println!("thread #2 says: {}", greeting);
    });
});
```

當以這種方式利用自動加入時，請注意，如果任何自動加入的執行緒出現恐慌，`thread::scope()` 將出現恐慌。

你可能已經註意到作用域執行緒現在只接受一個參數，它只是對 `s` 的另一個引用。由於 `s` 存在於作用域內，我們不能直接借用它。使用傳遞的參數來生成巢狀執行緒：

```rust
thread::scope(|s| {
    s.spawn(|s| {
        s.spawn(|_| {
            println!("I belong to the same `thread::scope()` as my parent thread")
        });
    });
});
```

# 參考級解釋
[參考級解釋]: #reference-level-explanation

我們在 `std::thread` 模組中新增兩個新的型別：

```rust
struct Scope<'env> {}
struct ScopedJoinHandle<'scope, T> {}
```

生命週期 `'env` 代表作用域外的環境，而 `'scope` 代表作用域本身。更準確地說，作用域外的所有內容都比 `'env` 和 `'scope` 內的所有內容都長。生命週期的關係是：

```
'variables_outside: 'env: 'scope: 'variables_inside
```

接下來，我們需要 `scoped()` 跟 `spawn()` 函式：

```rust
fn scope<'env, F, T>(f: F) -> T
where
    F: FnOnce(&Scope<'env>) -> T;

impl<'env> Scope<'env> {
    fn spawn<'scope, F, T>(&'scope self, f: F) -> ScopedJoinHandle<'scope, T>
    where
        F: FnOnce(&Scope<'env>) -> T + Send + 'env,
        T: Send + 'env;
}
```

這就是作用域執行緒的要點，真的。

現在我們只需要再做兩件事來使 API 完整。首先，`ScopedJoinHandle` 等同於 `JoinHandle`，但與 `'scope` 生命週期掛勾，所以它將有同樣的方法。第二，執行緒生成器需要能夠在一個作用域内生成執行緒。

```rust
impl<'scope, T> ScopedJoinHandle<'scope, T> {
    fn join(self) -> Result<T>;
    fn thread(&self) -> &Thread;
}

impl Builder {
    fn spawn_scoped<'scope, 'env, F, T>(
        self,
        &'scope Scope<'env>,
        f: F,
    ) -> io::Result<ScopedJoinHandle<'scope, T>>
    where
        F: FnOnce(&Scope<'env>) -> T + Send + 'env,
        T: Send + 'env;
}
```

# 缺點
[缺點]: #drawbacks

作用域執行緒的主要缺點是使標準函式庫有點大。

# 基本原理和替代方案
[基本原理和替代方案]: #rationale-and-alternatives

* 將作用域執行緒保留在外部 crates 之中。

  將他們放在標準函式庫有幾個優點：

  * 這是一個非常常見和實用的工具，非常適合學習、測試和探索性程式設計。每個學習 Rust 的人都會在某個時候遇到借用和執行緒的互動。有一個非常重要的教訓是執行緒實際上**可以**借用局部變數，但標準函式庫並沒有反映這一點。

  * 有些人可能會爭辯說我們應該完全不鼓勵使用執行緒，而是將人們指向像 Rayon 和 Tokio 這樣的執行器。但是，`thread::spawn()` 需要 F: `F: 'static` 並且無法繞過它，這感覺就像是標準函式庫中缺少的部分。

  * 實現作用域執行緒非常難處理，因此最好有標準函式庫提供一個可靠的解決方案。

  * 官方文檔和書籍中有許多範例可以透過作用域執行緒進行簡化。

  * 作用域執行緒通常是比 `thread::spawn()` 更好的默認值，因為它們確保生成的執行緒被連接並且不會意外 “洩漏”。這有時是單元測試中的一個問題，如果單元測試產生執行緒並忘記加入它們，“迷途” 的執行緒可能會積累。

  * 使用者一直在 IRC 和論壇上詢問作用域執行緒。將它們作為 `std::thread` 中的“祝福”模式對每個人都有好處。

* 從 `scope` 回傳一個 `Result`，包括所有捕獲的恐慌。

  * 這很快變得複雜，因為多個執行緒可能已經恐慌。回傳 `Vec` 或其他恐慌的集合並不總是最有用的介面，而且通常是不必要的。如果使用者想要處理它們，在 `ScopedJoinHandle` 上顯式使用 `.join()` 來處理恐慌是最靈活且最有效的方法。

* 不要將 `&Scope` 參數傳遞給執行緒。

  * 假如使用 `scope.spawn(|| ..)` 而不是 `scope.spawn(|scope| ..)`，則需要 `move` 關鍵字 (`scope.spawn(move || ..)`)，如果你想在 closure 內使用作用域，將變得不符合人因工程。


# 現有技術
[現有技術]: #prior-art

自 Rust 1.0 以來，Crossbeam 就有了[作用域執行緒](https://docs.rs/crossbeam/0.7.1/crossbeam/thread/index.html)。

Crossbeam 的作用域執行緒有兩種設計。舊的是在 `thread::scoped()` 被刪除後，我們想在 Rust 1.0 時代有一個合理的替代方案。新的則是在去年的大改版之中。

* 舊: https://docs.rs/crossbeam/0.2.12/crossbeam/fn.scope.html
* 新: https://docs.rs/crossbeam/0.7.1/crossbeam/fn.scope.html

新舊作用域執行緒之間存在一些差異：
1. `scope()` 現在從子執行緒傳播未處理的恐慌。在舊的設計中，恐慌被默默地忽略了。使用者仍然可以透過手動操作 `ScopedJoinHandle` 來處理恐慌。

2. 傳遞給 `Scope::spawn()` 的 closure 現在需要一個 `&Scope<'env>` 參數，該參數允許生成巢狀執行緒，這在舊設計中是不可能的。 Rayon 類似地傳遞了對子任務的引用。

3. 我們刪除了 `Scope::defer()`，因為它不是真的有用，有錯誤，並且有不明顯的行為。

4. `ScopedJoinHandle` 在 `'scope` 上進行了參數化，以防止它逃離作用域。

Rayon 也有作用域，但它們在不同的抽象級別上工作——Rayon 產生任務而不是執行緒。它的 API 與本 RFC 中提出的 API 相同。

# 未解決的問題
[未解決的問題]: #unresolved-questions

這個概念可以延伸到異步嗎？會有任何行為或 API 差異嗎？

# 未來的可能性
[未來的可能性]: #future-possibilities

在未来，我們也可以有一個像 Rayon 那樣的執行緒池，可以生成作用域内的任務。