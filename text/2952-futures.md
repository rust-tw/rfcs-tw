- Feature Name: `futures_api`
- Start Date: 2018-11-09
- RFC PR: [rust-lang/rfcs#2592](https://github.com/rust-lang/rfcs/pull/2592)
- Rust Issue: [rust-lang/rust#59113](https://github.com/rust-lang/rust/issues/59113)
- Translators: [[@FizzyElt](https://github.com/FizzyElt)]
- Commit: [The commit link this page based on](https://github.com/rust-lang/rfcs/blob/a31e5941133de5935b97b5d3e09b921b6e796abb/text/2592-futures.md)
- Updated 2022-08-13

# 概要
[概要]: #summary

此 RFC 提議替一等公民語法 `async`/`await` 穩定相關函式庫組件。特別是，它將穩定：

- `std` 中任務系統所有的 API，例如 `std::task::*`。
- 核心 `Future` API，例如 `core::future::Future` 和 `std::future::Future`。

它**不提議**穩定任何 `async`/`await` 語法本身，該語法自有另外的提議。它也不包括已在[別處](https://github.com/rust-lang/rust/issues/55766)提出的 `Pin` API 的穩定性。

這是早期 [futures RFC](https://github.com/rust-lang/rfcs/pull/2418) 的修訂和精簡版本，該 RFC 已暫緩，直到 nightly 版本獲得更多經驗之後。

[pin]: https://github.com/rust-lang/rfcs/pull/2349
[companion RFC]: https://github.com/rust-lang/rfcs/pull/2394

# 動機
[動機]: #motivation

## 為何 `Future` 要在 `std` 之中？

此 RFC 的核心動機是穩定 `async`/`await` 語法的支援機制。語法本身是在（已經合併的）[配套 RFC](https://github.com/rust-lang/rfcs/pull/2394) 中提出的，並且有一篇[部落格文章](https://aturon.github.io/tech/2018/04/24/async-borrowing/)更詳細介紹它的重要性。

與 closures 一樣，`async` 語法涉及到生成一個匿名型別，該型別實作一個關鍵特徵：`Future`。因為 `async`/`await` 需要語言層級的支援，所以底層特徵也必須是標準函式庫的一部分。因此，這個 RFC 的目標是穩定這個 `Future` 特徵和它所依賴的型別。這是我們能夠穩定 `async`/`await` 本身之前所需的最後一步。

## 這一步是如何融入更大的藍圖之中？


`async`/`await` 語法是 Rust 中最渴求的特性之一，它將對整個生態系產生重大影響。自 2018 年 5 月下旬以來，該語法及此處描述的 API 已可於 nightly 版本使用，並被大量採用。

穩定此設計的 futures API 部分使函式庫更容易在穩定的 Rust 上工作並無縫支援在 nightly 版本使用 `async`/`await`。它還允許我們完成關於 API 部分的設計討論，並專注於在 `async` 語法穩定之前剩下的幾個問題。

# 歷史背景

這些被提議要穩定的 API 有著悠久的歷史：

- `Future` 特徵從 futures crate 開始； [0.1 於 2016 年 8 月發布](https://aturon.github.io/tech/2016/08/11/futures/)。該版本確立了「任務/輪詢」模型的核心思想，以及此處保留的 API 的許多其他方面。 0.1 系列持續在整個 Rust 生態系和正式系統中大量使用。

- 2018 年初，隨著 `async`/`await` 的工作開始，futures 團隊建立了一個 RFC 流程並編寫了幾個 [RFC](https://github.com/rust-lang-nursery/futures-rfcs/pulls?q=is%3Apr+is%3Aclosed)，以根據長期的社群回饋對核心 API 進行修訂。這些 RFC 最終產生了 [0.2le](https://aturon.github.io/tech/2018/02/27/futures-0-2-RC/) 版本，並於 4 月[釋出](https://aturon.github.io/tech/2018/04/06/futures2/)。

- 在同一時期，@withoutboats 在支援 `async` 區塊內借用的 pinning API 上的工作已經[完成](https://boats.gitlab.io/blog/post/2018-04-06-async-await-final/)。[pinning API](https://github.com/rust-lang/rfcs/pull/2349) 顛覆了過往一切，可在不使 future 的核心 API 成為 unsafe 前提下，支援 borrowing-across-yield。

- 2018 年 4 月，一對 RFC 正式提出了 `async`/`await` 語法以及 futures API 的進一步修訂（以利用 pinning API 帶來的好處）；後者經歷了許多修訂，包括一個新的 [RFC](https://github.com/rust-lang/rfcs/pull/2418)。最終，語法 [RFC](https://github.com/rust-lang/rfcs/pull/2394#issuecomment-387550523) 在 5 月被合併，而 API RFC 被關閉，進一步的設計疊代將在 nightly 版本進行，隨後是一個穩定的 RFC：這個！

- 這些 API 在 5 月底[加入到 `std`](https://github.com/rust-lang/rust/pull/51263)。

- 從那時起，隨著我們從 API 獲得的經驗，語法、`std` API 和 futures 0.3 crate 都在同步發展。這種體驗的主要驅動者是 Google 的 Fuchsia 專案，該專案在作業系統設置中大規模使用所有這些功能。

- 最近的修訂是在 8 月，其中涉及一些關於如何使 `Pin` API 有更清晰的[見解](https://boats.gitlab.io/blog/post/rethinking-pin/)。這些 API 已被[提議要穩定](https://github.com/rust-lang/rust/issues/55766)，以及它們作為 [`self` 型別的用途](https://github.com/rust-lang/rust/issues/55786)。

- 有多個相容層可以同時使用 futures 0.1 和 0.3。這很重要，因為它允許現有生產程式碼的**增量**遷移。

自從最初的 futures 0.3 發布以來，除了上面提到的改進之外，核心 `Future` 特徵和任務系統的變化相對較小。實際的 `Future` 特徵基本上與 4 月時相同。

# 教學式解說
[教學式解說]: #guide-level-explanation

`Future` 特徵代表了一個非同步和惰性計算，它最終可能會產生一個最終值，但不以阻塞當前執行緒為前提來達成。

Future 可以通過 `async` 區塊或 `async` 函式來創建，例如
```rust
async fn read_frame(socket: &TcpStream) -> Result<Frame, io::Error> { ... }
```

This `async` function, when invoked, produces a future that represents the
completion of reading a frame from the given socket. The function signature
is equivalent to:
這個 `async` 函式，當被呼叫時，產生一個 future，代表從给定的 socket 中讀取一個幀的完成。函式簽名等同於：

```rust
fn read_frame<'sock>(socket: &'sock TcpStream)
    -> impl Future<Output = Result<Frame, io::Error>> + 'sock;
```

其他非同步函數可以 **await** 這個 future; 有關完整詳細信息，請參閱[隨附的 RFC](https://github.com/rust-lang/rfcs/pull/2394)。

除了 `async fn` 定義，future 還可以使用適配器來創建，就像 `Iterator` 一樣。最初，這些適配器將完全由「out of tree」提供，但最终它們將帶入標準函式庫。

最終非同步計算以**任務**的形式執行，相當於輕量級執行緒。 **executor** 提供了從 `()` 生成的 `Future` 創建任務的能力。執行器將固定 `Future` 並對其進行 `poll`，直到為其創建的任務完成。

executor 的實作以合作的方式調度它擁有的任務。是否使用一個或多個作業系統執行緒以及可以在其之上並行生成多少任務取決於 executor 的實作。一些 executor 的實作可能只能驅動單個 `Future` 完成，而另一些則可以提供動態接受新 `Future` 的能力，這些 Future 是在任務中被驅動完成。

該 RFC 不包含任何 executor 的定義。它僅以 API 的形式定義了 executors、tasks 和 `Future` 之間的整合，允許任務請求再次被調度。 `task` 模組提供了這些 API，在手動實作 `Future` 或 executor 時需要這些 API。

# 技術文件式解說
[技術文件式解說]: #reference-level-explanation

## `core::task` 模組

Rust 中非同步運算的基本機制是**任務**，它們是輕量級的執行緒；許多任務可以協同調度到單個作業系統執行緒上。

為了執行這種協作調度，我們使用了一種有時被稱為「[trampoline](https://en.wikipedia.org/wiki/Trampoline_(computing))」的技術。當一個任務需要阻塞等待某個事件時，它會保存一個物件，允許它稍後再次被調度並**返回**到運行它的執行器，然後它可以執行另一個任務。隨後的喚醒將任務放回就緒任務的執行者隊列中，就像作業系統中的執行緒調度程序一樣。

嘗試完成一個任務（或其中的非同步值）稱為**輪詢**，並且總是返回一個 `Poll` 值：

```rust
/// Indicates whether a value is available, or if the current task has been
/// scheduled for later wake-up instead.
#[derive(Copy, Clone, Debug, PartialEq)]
pub enum Poll<T> {
    /// Represents that a value is immediately ready.
    Ready(T),

    /// Represents that a value is not ready yet.
    ///
    /// When a function returns `Pending`, the function *must* also
    /// ensure that the current task is scheduled to be awoken when
    /// progress can be made.
    Pending,
}
```

當一個任務返回 `Poll::Ready` 時，執行器知道該任務已經完成並且可以被刪除。

### 喚醒

如果 future 在執行期間無法直接完成並返回 `Pending`，則它需要一種方法來稍後通知執行器它需要再次輪詢以取得進展。

此功能是透過一組 `Waker` 型別提供的。

`Waker` 是作為參數傳遞給 `Future::poll` 調用的物件，並且可以透過這些 `Futures` 的實現來儲存。每當一個 `Future` 需要再次被輪詢時，它可以使用喚醒器的 `wake` 方法來通知執行器擁有 `Future` 的任務應該被再次調度和執行。

RFC 定義了一個具體的 `Waker` 型別，`Futures` 和非同步函式的實作者將與之互動。此型別定義了一個 `wake(&self)` 方法，用於安排與 `Waker` 關聯的任務再次輪詢。

再次調度任務的機制取決於驅動任務的執行器。喚醒執行器的可能方法包含：
- 如果執行器在條件變數上被阻塞，則需要通知條件變數。
- 如果執行器在諸如 `select` 之類的系統調用上被阻塞，它可能需要被諸如 `write` 管道之類的系統調用喚醒。
- 如果執行器的執行緒被放置，喚醒呼叫需要將其取消放置。

為了讓執行器實現自定義喚醒行為，`Waker` 型別包含一個稱作 `RawWaker` 的型別，它由指向自定義可喚醒物件的指針和對其提供 `clone`、`wake` 和 `drop` 函式的虛擬函數指針表 (vtable) 的引用組成底層可喚醒物件。

選擇這種機制有利於特徵物件，因為它允許更靈活的記憶體管理方案。 `RawWaker` 可以單純根據全域函式和狀態、在引用計數物件之上或以其他方式實現。這種策略還可以更容易地提供不同的 vtable 函式，這些函式將執行不同的行為，儘管引用了相同底層可喚醒的物件型別。

這些 `Waker` 型別之間的關係在以下定義中進行了概述：
```rust
/// A `RawWaker` allows the implementor of a task executor to create a `Waker`
/// which provides customized wakeup behavior.
///
/// It consists of a data pointer and a virtual function pointer table (vtable) that
/// customizes the behavior of the `RawWaker`.
#[derive(PartialEq)]
pub struct RawWaker {
    /// A data pointer, which can be used to store arbitrary data as required
    /// by the executor. This could be e.g. a type-erased pointer to an `Arc`
    /// that is associated with the task.
    /// The value of this field gets passed to all functions that are part of
    /// the vtable as first parameter.
    pub data: *const (),
    /// Virtual function pointer table that customizes the behavior of this waker.
    pub vtable: &'static RawWakerVTable,
}

/// A virtual function pointer table (vtable) that specifies the behavior
/// of a `RawWaker`.
///
/// The pointer passed to all functions inside the vtable is the `data` pointer
/// from the enclosing `RawWaker` object.
#[derive(PartialEq, Copy, Clone)]
pub struct RawWakerVTable {
    /// This function will be called when the `RawWaker` gets cloned, e.g. when
    /// the `Waker` in which the `RawWaker` is stored gets cloned.
    ///
    /// The implementation of this function must retain all resources that are
    /// required for this additional instance of a `RawWaker` and associated
    /// task. Calling `wake` on the resulting `RawWaker` should result in a wakeup
    /// of the same task that would have been awoken by the original `RawWaker`.
    pub clone: unsafe fn(*const ()) -> RawWaker,

    /// This function will be called when `wake` is called on the `Waker`.
    /// It must wake up the task associated with this `RawWaker`.
    pub wake: unsafe fn(*const ()),

    /// This function gets called when a `RawWaker` gets dropped.
    ///
    /// The implementation of this function must make sure to release any
    /// resources that are associated with this instance of a `RawWaker` and
    /// associated task.
    pub drop_fn: unsafe fn(*const ()),
}

/// A `Waker` is a handle for waking up a task by notifying its executor that it
/// is ready to be run.
///
/// This handle encapsulates a `RawWaker` instance, which defines the
/// executor-specific wakeup behavior.
///
/// Implements `Clone`, `Send`, and `Sync`.
pub struct Waker {
    waker: RawWaker,
}

impl Waker {
    /// Wake up the task associated with this `Waker`.
    pub fn wake(&self) {
        // The actual wakeup call is delegated through a virtual function call
        // to the implementation which is defined by the executor.
        unsafe { (self.waker.vtable.wake)(self.waker.data) }
    }

    /// Returns whether or not this `Waker` and other `Waker` have awaken the same task.
    ///
    /// This function works on a best-effort basis, and may return false even
    /// when the `Waker`s would awaken the same task. However, if this function
    /// returns `true`, it is guaranteed that the `Waker`s will awaken the same task.
    ///
    /// This function is primarily used for optimization purposes.
    pub fn will_wake(&self, other: &Waker) -> bool {
        self.waker == other.waker
    }

    /// Creates a new `Waker` from `RawWaker`.
    ///
    /// The method cannot check whether `RawWaker` fulfills the required API
    /// contract to make it usable for `Waker` and is therefore unsafe.
    pub unsafe fn new_unchecked(waker: RawWaker) -> Waker {
        Waker {
            waker: waker,
        }
    }
}

impl Clone for Waker {
    fn clone(&self) -> Self {
        Waker {
            waker: unsafe { (self.waker.vtable.clone)(self.waker.data) },
        }
    }
}

impl Drop for Waker {
    fn drop(&mut self) {
        unsafe { (self.waker.vtable.drop_fn)(self.waker.data) }
    }
}
```

`Waker` 必須滿足以下要求：
- 它們必須是可以克隆的。
- 如果 `Waker` 的所有實體都已被刪除，並且它們的關聯任務已被驅動完成，則必須釋放為該任務分配的所有資源。
- 即使關聯的任務已經被驅動完成，在 `Waker` 上調用 `wake()` 也必須是安全的。
- `Waker::wake()` 必須喚醒執行器，即使它是從任意執行緒調用的。

因此，實例化 `RawWaker` 的執行程器必須確保滿足這些要求。

## `core::future` 模組

有了上述所有任務的基礎設施，定義 `Future` 就很簡單了：

```rust
pub trait Future {
    /// The type of value produced on completion.
    type Output;

    /// Attempt to resolve the future to a final value, registering
    /// the current task for wakeup if the value is not yet available.
    ///
    /// # Return value
    ///
    /// This function returns:
    ///
    /// - [`Poll::Pending`] if the future is not ready yet
    /// - [`Poll::Ready(val)`] with the result `val` of this future if it
    ///   finished successfully.
    ///
    /// Once a future has finished, clients should not `poll` it again.
    ///
    /// When a future is not ready yet, `poll` returns `Poll::Pending` and
    /// stores a clone of the [`Waker`] to be woken once the future can
    /// make progress. For example, a future waiting for a socket to become
    /// readable would call `.clone()` on the [`Waker`] and store it.
    /// When a signal arrives elsewhere indicating that the socket is readable,
    /// `[Waker::wake]` is called and the socket future's task is awoken.
    /// Once a task has been woken up, it should attempt to `poll` the future
    /// again, which may or may not produce a final value.
    ///
    /// Note that on multiple calls to `poll`, only the most recent
    /// [`Waker`] passed to `poll` should be scheduled to receive a
    /// wakeup.
    ///
    /// # Runtime characteristics
    ///
    /// Futures alone are *inert*; they must be *actively* `poll`ed to make
    /// progress, meaning that each time the current task is woken up, it should
    /// actively re-`poll` pending futures that it still has an interest in.
    ///
    /// The `poll` function is not called repeatedly in a tight loop-- instead,
    /// it should only be called when the future indicates that it is ready to
    /// make progress (by calling `wake()`). If you're familiar with the
    /// `poll(2)` or `select(2)` syscalls on Unix it's worth noting that futures
    /// typically do *not* suffer the same problems of "all wakeups must poll
    /// all events"; they are more like `epoll(4)`.
    ///
    /// An implementation of `poll` should strive to return quickly, and must
    /// *never* block. Returning quickly prevents unnecessarily clogging up
    /// threads or event loops. If it is known ahead of time that a call to
    /// `poll` may end up taking awhile, the work should be offloaded to a
    /// thread pool (or something similar) to ensure that `poll` can return
    /// quickly.
    ///
    /// # Panics
    ///
    /// Once a future has completed (returned `Ready` from `poll`),
    /// then any future calls to `poll` may panic, block forever, or otherwise
    /// cause bad behavior. The `Future` trait itself provides no guarantees
    /// about the behavior of `poll` after a future has completed.
    ///
    /// [`Poll::Pending`]: ../task/enum.Poll.html#variant.Pending
    /// [`Poll::Ready(val)`]: ../task/enum.Poll.html#variant.Ready
    /// [`Waker`]: ../task/struct.Waker.html
    /// [`Waker::wake`]: ../task/struct.Waker.html#method.wake
    fn poll(self: Pin<&mut Self>, waker: &Waker) -> Poll<Self::Output>;
}
```

這裡的大部分解釋都遵循我們已經說過的關於任務系統的內容。一個轉折是 `Pin` 的使用，這使得可以在不同的 `poll` 調用中保留借用資訊（即「borrowing over yield
points」）。固定機制在介紹它的 [RFC](https://github.com/rust-lang/rfcs/pull/2349) 和有關最新修訂的[部落格文章](https://boats.gitlab.io/blog/post/rethinking-pin/)中進行了解釋。

## 與 futures 0.1 的關係

上面歷史背景部分概述的各種討論涵蓋了從 futures 0.1 到這些 API 的演進。但是，簡而言之，有三個主要轉變：

- 使用 `Pin<&mut self>` 而不單純是 `&mut self`，這是支援借用 `async` 區塊所必需的。 `Unpin` 標記特徵可用於在手動撰寫 futures 時恢復類似於 futures 0.1 的人因工程和安全性。

- 從 `Future` 中刪除**內置**錯誤，以支援 `Future` 在可能失敗時返回 `Result`。 futures 0.3 crate 提供了一個 `TryFuture` 特徵，該特徵在 `Result` 中處理，以便在使用 `Result`-producer futures 時提供更好的人因工程。刪除錯誤型別已在之前的執行緒中討論過，但最重要的理由是為 `async fn` 提供一個正交的、組合性的語義，以反映正常的 `fn`，而不是採用特定的錯誤處理風格。

- 顯式傳遞 `Waker`，而不是將其儲存在執行緒本地存儲中。自 futures 0.1 發布以來，這一直是一個備受爭議的問題，該 RFC 並不打算重新討論這個問題，但總而言之，主要優點是 (1) 在使用手動 futures（與 `async` 區塊相反）時，它更容易分辨哪裡需要環境任務，並且 (2) `no_std` 相容性明顯更好。

為了彌補 futures 0.1 和 0.3 之間的差距，有幾個相容性鋪墊，包括一個內置於 futures crate 本身的，您可以簡單地透過使用 `.compat()` 組合器在兩者之間切換。這些相容層使現有生態系可以順利地使用新的 futures API，並使大型程式的漸進式轉換成為可能。

# Rationale, drawbacks, and alternatives

This RFC is one of the most substantial additions to `std` proposed since
1.0. It commits us to including a particular task and polling model in the
standard library, and ties us to `Pin`.

So far we've been able to push the task/polling model into virtually every niche
Rust wishes to occupy, and the main downside has been, in essence, the lack of
async/await syntax (and
the
[borrowing it supports](https://aturon.github.io/tech/2018/04/24/async-borrowing/)).

This RFC does not attempt to provide a complete introduction to the task model
that originated with the futures crate. A fuller account of the design rationale
and alternatives can be found in the following two blog posts:

- [Zero-cost futures in Rust](https://aturon.github.io/tech/2016/08/11/futures/)
- [Designing futures for Rust](https://aturon.github.io/tech/2016/09/07/futures-design/)

To summarize, the main alternative model for futures is a callback-based approach,
which was attempted for several months before the current approach was discovered.
In our experience, the callback approach suffered from several drawbacks in Rust:

- It forced allocation almost everywhere, and hence was not compatible with no_std.
- It made cancellation *extremely* difficult to get right, whereas with the
  proposed model it's just "drop".
- Subjectively, the combinator code was quite hairy, while with the task-based model
  things fell into place quickly and easily.

Some additional context and rationale for the overall async/await project is
available in the [companion RFC].

For the remainder of this section, we'll dive into specific API design questions
where this RFC differs from futures 0.2.

## Rationale, drawbacks and alternatives for removing built-in errors

There are an assortment of reasons to drop the built-in error type in the main
trait:

- **Improved type checking and inference**. The error type is one of the biggest
  pain points when working with futures combinators today, both in trying to get
  different types to match up, and in inference failures that result when a
  piece of code cannot produce an error. To be clear, many of these problems
  will become less pronounced when `async` syntax is available.

- **Async functions**. If we retain a built-in error type, it's much less clear
  how `async fn` should work: should it always require the return type to be a
  `Result`? If not, what happens when a non-`Result` type is returned?

- **Combinator clarity**. Splitting up the combinators by whether they rely on
  errors or not clarifies the semantics. This is *especially* true for streams,
  where error handling is a common source of confusion.

- **Orthogonality**. In general, producing and handling errors is separable from
  the core polling mechanism, so all things being equal, it seems good to follow
  Rust's general design principles and treat errors by *composing* with `Result`.

All of that said, there are real downsides for error-heavy code, even with
`TryFuture`:

- An extra import is needed (obviated if code imports the futures prelude, which
  we could perhaps more vocally encourage).

- It can be confusing for code to *bound* by one trait but *implement* another.

The error handling piece of this RFC is separable from the other pieces, so the
main alternative would be to retain the built-in error type.

## Rationale, drawbacks and alternatives to the core trait design (wrt `Pin`)

Putting aside error handling, which is orthogonal and discussed above, the
primary other big item in this RFC is the move to `Pin` for the core polling
method, and how it relates to `Unpin`/manually-written futures. Over the course
of RFC discussions, we've identified essentially three main approaches to this
question:

- **One core trait**. That's the approach taken in the main RFC text: there's
  just a single core `Future` trait, which works on `Pin<&mut Self>`. Separately
  there's a `poll_unpin` helper for working with `Unpin` futures in manual
  implementations.

- **Two core traits**. We can provide two traits, for example `MoveFuture` and
  `Future`, where one operates on `&mut self` and the other on `Pin<&mut Self>`.
  This makes it possible to continue writing code in the futures 0.2 style,
  i.e. without importing `Pin`/`Unpin` or otherwise talking about pins. A
  critical requirement is the need for interoperation, so that a `MoveFuture`
  can be used anywhere a `Future` is required. There are at least two ways to
  achieve such interop:

  - Via a blanket impl of `Future` for `T: MoveFuture`. This approach currently
    blocks some *other* desired impls (around `Box` and `&mut` specifically),
    but the problem doesn't appear to be fundamental.

  - Via a subtrait relationship, so that `T: Future` is defined essentially as
    an alias for `for<'a> Pin<&mut 'a T>: MoveFuture`. Unfortunately, such
    "higher ranked" trait relationships don't currently work well in the trait
    system, and this approach also makes things more convoluted when
    implementing `Future` by hand, for relatively little gain.

The drawback of the "one core trait" approach taken by this RFC is its ergonomic
hit when writing moveable futures by hand: you now need to import `Pin` and
`Unpin`, invoke `poll_unpin`, and impl `Unpin` for your types. This is all
pretty mechanical, but it's a pain. It's possible that improvements in `Pin`
ergonomics will obviate some of these issues, but there are a lot of open
questions there still.

On the other hand, a two-trait approach has downsides as well. If we *also*
remove the error type, there's a combinatorial explosion, since we end up
needing `Try` variants of each trait (and this extends to related traits, like
`Stream`, as well). More broadly, with the one-trait approach, `Unpin` acts as a
kind of "independent knob" that can be applied orthogonally from other concerns;
with the two-trait approach, it's "mixed in". And both of the two-trait
approaches run up against compiler limitations at the moment, though of course
that shouldn't be taken as a deciding factor.

**The primary reason this RFC opts for the one-trait approach is that it's the
conservative, forward-compatible option, and has proven itself in practice**.
It's possible to add `MoveFuture`, together with a blanket impl, at any point in the future.
Thus, starting with just the single `Future` trait as proposed in this RFC keeps our options
maximally open while we gain experience.

## Rationale, drawbacks and alternatives to the wakeup design (`Waker`)

Previous iterations of this proposal included a separate wakeup type,
`LocalWaker`, which was `!Send + !Sync` and could be used to implement
optimized executor behavior without requiring atomic reference counting
or atomic wakeups. However, in practice, these same optimizations are
available through the use of thread-local wakeup queues, carrying IDs
rather than pointers to wakeup objects, and tracking an executor ID or
thread ID to perform a runtime assertion that a `Waker` wasn't sent across
threads. For a simple example, a single thread-locked executor with zero
atomics can be implemented as follows:

```rust
struct Executor {
    // map from task id (usize) to task
    tasks: Slab<Task>,
    // list of woken tasks to poll
    work_queue: VecDeque<usize>,
}

thread_local! {
  pub static EXECUTOR: RefCell<Option<Executor>> = ...;
}

static VTABLE: &RawWakerVTable = &RawWakerVTable {
    clone: |data: *const ()| RawWaker { data, vtable: VTABLE, },
    wake: |data: *const ()| EXECUTOR.borrow_mut().as_mut().expect(...).work_queue.push(data as usize),
    drop,
};
```

While this solution gives inferior error messages to the `LocalWaker` approach
(since it can't panic until `wake` occurs on the wrong thread, rather than
panicking when `LocalWaker` is transformed into a `Waker`), it dramatically
simplifies the user-facing API by de-duplicating the `LocalWaker` and `Waker`
types.

In practice, it's also likely that the most common executors in the Rust
ecosystem will continue to be multithreaded-compatible (as they are today),
so optimizing for the ergonomics of this case is prioritized over better
error messages in the more heavily specialized case.

# Prior art
[prior-art]: #prior-art

There is substantial prior art both with async/await notation and with futures
(aka promises) as a basis. The proposed futures API was influenced by Scala's
futures in particular, and is broadly similar to APIs in a variety of other
languages (in terms of the adapters provided).

What's more unique about the model in this RFC is the use of tasks, rather than
callbacks. The RFC author is not aware of other *futures* libraries using this
technique, but it is a fairly well-known technique more generally in functional
programming. For a recent example,
see
[this paper](https://www.microsoft.com/en-us/research/wp-content/uploads/2011/01/monad-par.pdf) on
parallelism in Haskell. What seems to be perhaps new with this RFC is the idea
of melding the "trampoline" technique with an explicit, open-ended task/wakeup
model.

# Unresolved questions
[unresolved]: #unresolved-questions

None at the moment.
