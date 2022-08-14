- Feature Name: `futures_api`
- Start Date: 2018-11-09
- RFC PR: [rust-lang/rfcs#2592](https://github.com/rust-lang/rfcs/pull/2592)
- Rust Issue: [rust-lang/rust#59113](https://github.com/rust-lang/rust/issues/59113)
- Translators: [[@FizzyElt](https://github.com/FizzyElt)]
- Commit: [The commit link this page based on](https://github.com/rust-lang/rfcs/blob/a31e5941133de5935b97b5d3e09b921b6e796abb/text/2592-futures.md)
- Updated 2022-08-13

# 概要
[概要]: #summary

該 RFC 建議為一級的 `async`/`await` 穩定相關函式庫組件。特別是，它將穩定：

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


`async`/`await` 語法是 Rust 中最渴求的特性之一，它將對整個生態系產生重大影響。以及此處描述的 API 已在 nightly 版本可用，並於 2018 年 5 月下旬開始投入使用。

穩定此設計的 futures API 部分使函式庫更容易在穩定的 Rust 上工作並無縫支援在 nightly 版本使用 `async`/`await`。它還允許我們完成關於 API 部分的設計討論，並專注於在 `async` 語法穩定之前剩下的幾個問題。

# 歷史背景

這些被提議要穩定的 API 有著悠久的歷史：

- `Future` 特徵從 futures crate 開始； [0.1 於 2016 年 8 月發布](https://aturon.github.io/tech/2016/08/11/futures/)。該版本確立了「任務/輪詢」模型的核心思想，以及此處保留的 API 的許多其他方面。 0.1 系列持續在整個 Rust 生態系和正式系統中大量使用。

- 2018 年初，隨著 `async`/`await` 的工作開始，futures 團隊建立了一個 RFC 流程並編寫了幾個 [RFC](https://github.com/rust-lang-nursery/futures-rfcs/pulls?q=is%3Apr+is%3Aclosed)，以根據長期的社群回饋對核心 API 進行修訂。這些 RFC 最終產生了 [0.2le](https://aturon.github.io/tech/2018/02/27/futures-0-2-RC/) 版本，並於 4 月[釋出](https://aturon.github.io/tech/2018/04/06/futures2/)。

- 在同一時期，@withoutboats 在支援 `async` 區塊內借用的 pinning API 上的工作已經[完成](https://boats.gitlab.io/blog/post/2018-04-06-async-await-final/)。[pinning API](https://github.com/rust-lang/rfcs/pull/2349) 改變了遊戲規則，使得支援 borrowing-across-yield 而不會使 future 的核心 API 變得不安全。

- 2018 年 4 月，一對 RFC 正式提出了 `async`/`await` 語法以及 futures API 的進一步修訂（以利用 pinning API 帶來的好處）；後者經歷了許多修訂，包括一個新的 [RFC](https://github.com/rust-lang/rfcs/pull/2418)。最終，語法 [RFC](https://github.com/rust-lang/rfcs/pull/2394#issuecomment-387550523) 在 5 月被合併，而 API RFC 被關閉，進一步的設計疊代將在 nightly 版本進行，隨後是一個穩定的 RFC：這個！

- 這些 API 在 5 月底[降臨到 `std`](https://github.com/rust-lang/rust/pull/51263)。

- 從那時起，隨著我們從 API 獲得的經驗，語法、`std` API 和 futures 0.3 crate 都在同步發展。這種體驗的主要驅動者是 Google 的 Fuchsia 專案，該專案在作業系統設置中大規模使用所有這些功能。

- 最近的修訂是在 8 月，其中涉及一些關於如何使 `Pin` API 有更清晰的[見解](https://boats.gitlab.io/blog/post/rethinking-pin/)。這些 API 已被[提議要穩定](https://github.com/rust-lang/rust/issues/55766)，以及它們作為 [`self` 型別的用途](https://github.com/rust-lang/rust/issues/55786)。

- 有多個相容層可以同時使用 futures 0.1 和 0.3。這很重要，因為它允許現有生產程式碼的**增量**遷移。

自從最初的 futures 0.3 發布以來，除了上面提到的改進之外，核心 `Future` 特徵和任務系統的變化相對較小。實際的 `Future` 特徵基本上與 4 月時相同。

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The `Future` trait represents an *asynchronous* and lazy computation that may
eventually produce a final value, but doesn't have to block the current thread
to do so.

Futures can be constructed through `async` blocks or `async` functions, e.g.,

```rust
async fn read_frame(socket: &TcpStream) -> Result<Frame, io::Error> { ... }
```

This `async` function, when invoked, produces a future that represents the
completion of reading a frame from the given socket. The function signature
is equivalent to:

```rust
fn read_frame<'sock>(socket: &'sock TcpStream)
    -> impl Future<Output = Result<Frame, io::Error>> + 'sock;
```

Other async functions can *await* this future; see the [companion
RFC] for full details.

In addition to `async fn` definitions, futures can be built using adapters, much
like with `Iterator`s. Initially these adapters will be provided entirely "out
of tree", but eventually they will make their way into the standard library.

Ultimately asynchronous computations are executed in the form of *tasks*,
which are comparable to lightweight threads. *executor*s provide the ability to
create tasks from `()`-producing `Future`s. The executor will pin the `Future`
and `poll` it until completion inside the task that it creates for it.

The implementation of an executor schedules the tasks it owns in a cooperative
fashion. It is up to the implementation of an executor whether one or more
operation system threads are used for this, as well as how many tasks can be
spawned on it in parallel. Some executor implementations may only be able to
drive a single `Future` to completion, while others can provide the ability to
dynamically accept new `Future`s that are driven to completion inside tasks.

This RFC does not include any definition of an executor. It merely defines the
interaction between executors, tasks and `Future`s, in the form of APIs
that allow tasks to request getting scheduled again.
The `task` module provides these APIs, which are required when manually implementing
`Future`s or executors.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## `core::task` module

The fundamental mechanism for asynchronous computation in Rust is *tasks*, which
are lightweight threads of execution; many tasks can be cooperatively scheduled
onto a single operating system thread.

To perform this cooperative scheduling we use a technique sometimes referred to
as a "trampoline". When a task would otherwise need to block waiting for some
event, instead it saves an object that allows it to get scheduled again later
and *returns* to the executor running it, which can then run another task.
Subsequent wakeups place the task back on the executors queue of ready tasks,
much like a thread scheduler in an operating system.

Attempting to complete a task (or async value within it) is called *polling*,
and always yields a `Poll` value back:

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

When a task returns `Poll::Ready`, the executor knows the task has completed and
can be dropped.

### Waking up

If a future cannot be directly fulfilled during execution and returns `Pending`,
it needs a way to later on inform the executor that it needs to get polled again
to make progress.

This functionality is provided through a set of `Waker` types.

`Waker`s are objects which are passed as a parameter to the `Future::poll` call,
and which can be stored by the implementation of those `Futures`s. Whenever a
`Future` has the need to get polled again, it can use the `wake` method of the
waker in order to inform the executor that the task which owns the `Future`
should get scheduled and executed again.

The RFC defines a concrete `Waker` type with which implementors of `Futures`
and asynchronous functions will interact. This type defines a `wake(&self)`
method which is used to schedule the task that is associated to the `Waker`
to be polled again.

The mechanism through which tasks get scheduled again depends on the executor
which is driving the task.
Possible ways of waking up an executor include:
- If the executor is blocked on a condition variable, the condition variable
  needs to get notified.
- If the executor is blocked on a system call like `select`, it might need
  to get woken up by a syscall like `write` to a pipe.
- If the executor's thread is parked, the wakeup call needs to unpark it.

To allow executors to implement custom wakeup behavior, the `Waker` type
contains a type called `RawWaker`, which consists of a  pointer
to a custom wakeable object and a reference to a virtual function
pointer table (vtable) which provides functions to `clone`, `wake`, and
`drop` the underlying wakeable object.

This mechanism is chosen in favor of trait objects since it allows for more
flexible memory management schemes. `RawWaker` can be implemented purely in
terms of global functions and state, on top of reference counted objects, or
in other ways. This strategy also makes it easier to provide different vtable
functions that will perform different behaviors despite referencing the same
underlying wakeable object type.

The relation between those `Waker` types is outlined in the following definitions:

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

`Waker`s must fulfill the following requirements:
- They must be cloneable.
- If all instances of a `Waker` have been dropped and their associated task had
  been driven to completion, all resources which had been allocated for the task
  must have been released.
- It must be safe to call `wake()` on a `Waker` even if the associated task has
  already been driven to completion.
- `Waker::wake()` must wake up an executor even if it is called from an arbitrary
  thread.

An executor that instantiates a `RawWaker` must therefore make sure that all
these requirements are fulfilled.

## `core::future` module

With all of the above task infrastructure in place, defining `Future` is
straightforward:

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

Most of the explanation here follows what we've already said about the task
system. The one twist is the use of `Pin`, which makes it possible to keep data
borrowed across separate calls to `poll` (i.e., "borrowing over yield
points"). The mechanics of pinning are explained
in [the RFC that introduced it](https://github.com/rust-lang/rfcs/pull/2349)
and the [blog post about t he latest revisions](https://boats.gitlab.io/blog/post/rethinking-pin/).

## Relation to futures 0.1

The various discussions outlined in the historical context section above cover the
path to these APIs from futures 0.1. But, in a nutshell, there are three major shifts:

- The use of `Pin<&mut self>` rather than just `&mut self`, which is necessary
to support borrowing withing `async` blocks. The `Unpin` marker trait can be used
to restore ergonomics and safety similar to futures 0.1 when writing futures by hand.

- Dropping *built in* errors from `Future`, in favor of futures returning a `Result`
when they can fail. The futures 0.3 crate provides a `TryFuture` trait that bakes
in the `Result` to provide better ergonomics when working with `Result`-producing futures.
Dropping the error type has been discussed in previous threads, but the most
important rationale is to provide an orthogonal, compositional semantics for `async fn`
that mirrors normal `fn`, rather than *also* baking in a particular style of
error handling.

- Passing a `Waker` explicitly, rather than stashing it in thread-local storage.
This has been a hotly debated issue since futures 0.1 was released, and this
RFC does not seek to relitigate it, but to summarize, the major advantages are (1)
when working with manual futures (as opposed to `async` blocks) it's much easier to
tell where an ambient task is required, and (2) `no_std` compatibility is
significantly smoother.

To bridge the gap between futures 0.1 and 0.3, there are several compatibility shims,
including one built into the futures crate itself, where you can shift between the two
simply by using a `.compat()` combinator. These compatibility layers make it possible
to use the existing ecosystem smoothly with the new futures APIs, and make it possible
to transition large code bases incrementally.

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
