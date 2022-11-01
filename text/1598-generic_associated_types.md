- Feature Name: generic_associated_types
- Start Date: 2016-04-29
- RFC PR: [rust-lang/rfcs#1598](https://github.com/rust-lang/rfcs/pull/1598)
- Rust Issue: [rust-lang/rust#44265](https://github.com/rust-lang/rust/issues/44265)
- Translators: [@wusyong](/ltc0FXDLS5S95FIi_AwVUg)
- Commit: [a7cd910](https://github.com/rust-lang/rfcs/commit/a7cd91048eea3d7ae83bec20446e62bad0c45381)
- Updated: 2022-11-02

# 總結
[summary]: #總結

讓型別建構子（type constructor）能與特徵（trait）相互關聯。在 Rust 使用者最想要的功能中，有個通常稱為「嗨爾卡因斗太普（higher-kinded types）」的功能，而這是達成這項目標的其中一步。關聯型別建構仔（associated type constructors）這項特定的功能可以解決嗨爾卡因斗其中一項最普遍的使用情境，比起其他形式的嗨爾卡因斗多型（polymorphism）更容易擴充至原有的型別系統，而且能兼容未來可能所導入的更複雜形式的嗨爾卡因斗多型。

# 動機
[motivation]: #動機

讓我們先考慮以下的特徵範例來展示可能的動機：

```rust
trait StreamingIterator {
    type Item<'a>;
    fn next<'a>(&'a mut self) -> Option<Self::Item<'a>>;
}
```

這樣的特徵是很實用的，它能提供一種疊代器（Iterator）所回傳的數值能附有一個生命週期，且能與傳至 `next` 的引用所擁有的生命週期綁定。此特徵最明顯的使用情境就是對一個向量（vector）所產生的疊代器，在每次疊代時都能產生重疊（overlapping）且可變（mutable）的子切片（subslice）。使用標準的 `Iterator` 介面的話，這樣的實作是無效的，因為每個切片必須活得與整個疊代器一樣長，而非只是透過 `next` 借用的時間而已。

這樣的特徵無法用現在的 Rust 表現出來，因為它需要依賴某種形式的嗨爾卡因斗多型。本 RFC 將擴充 Rust 來確保此特定形式的嗨爾卡因斗多型，也就是我們在此所稱作的關聯型別建構子。此功能有各種應用場景，但最主要的應用都還是與 `StreamingIterator` 類似：定義的特徵所持有的型別能附有一個生命週期，並與接收型別所借用的生命週期相互關聯。

# 設計細節
[design]: #設計細節

## 背景：什麼是卡因斗？

「嗨爾卡因斗太普」算是比較攏統的詞彙，容易與其他語言功能混淆，而造成我們想表達的概念不精確。所以本 RFC 將簡單介紹卡因斗的概念。卡因斗通常被稱為「型別的型別」，但這樣的解釋對於想理解的人幫助並不大，反而對已經了解這些概念的人才說得通。所以讓我們嘗試用型別比喻來了解卡因斗。

在型別完善的語言中，每個表達式（expression）都有個型別。許多表達式都有所謂的「基本型別」，也就是語言中的原生型別而且無法用其他型別表達。在 Rust 中，`bool`、`i64`、`usize` 與 `char` 都是基本型別的明顯例子。另一方面，也就會有型別會是由其他型別組成，函式就是屬於這樣的例子。讓我們看看下面這個簡單的函式：

```rust
fn not(x: bool) -> bool {
   !x
}
```

`not` 的型別為 `bool -> bool`（抱歉這邊開始會使用些與 Rust 不大一樣的語法），而  `not(true)` 的型別則是 `bool`。注意這樣的不同點正是了解嗨爾卡因斗的關鍵。

在卡因斗的分析中，`bool`、`char`、`bool -> bool` 等等型別都有 `type` 這種卡因斗。每個型別都有  `type` 卡因斗。然而 `type` 只是基本卡因斗，就像 `bool` 只是個基本型別，我們能寫出更複雜的卡因斗像是 `type -> type`，這樣的卡因斗其中一個範例就是 `Vec` 能接受一個型別然後產生出一個型別。而 `Vec` 的卡因斗與 `Vec<i32>` 的卡因斗（也就是 `type`）之間的差異，就相對應於 `not` 與 `not(true)` 之間的差異。另外注意 `Vec<T>` 的卡因斗和 `Vec<i32>` 一樣仍然是 `type`。盡管 `T` 是個型別參數，`Vec<T>` 還是接收了一個型別，就像 `not(x)` 的型別仍然是 `bool`，盡管 `x` 是個變數一樣。

Rust 相對不尋常的特色是它有**兩種**基本卡因斗，相較於許多語言處理嗨爾卡因斗時只會有一種卡因斗 `type`。Rust 的另一個基本卡因斗就是生命週期參數。如果你有個型別像是 `Foo<'a>`，`Foo` 的卡因斗就是 `lifetime -> type`。

而嗨爾卡因斗當然也可以接受多重引數，`Result` 的卡因斗就是 `type, type -> type`，而  `vec::Iter<'a, T>` 中的 `vec::Iter` 則會有 `lifetime, type -> type` 這樣的卡因斗。

嗨爾卡因斗的表達通常就稱為「型別運算子（type operators）」，運算出一個型別的型別運算子就稱作「型別建構子（type constructors）」。也有其他型別運算子能運算出其他型別運算子，而且還有更嗨爾的型別運算子能接收型別運算子作為引數，讓它們的卡因斗變成像是 `(type -> type) -> type`。本 RFC 將不會討論這些更進階的範疇。

本 RFC 更明確的目標就是要讓型別建構子可以與特徵相互關聯，就像你現在的特徵能與函式、型別與 consts 相互關聯一樣。型別建構子還有包含其他形式的多型，像是對型別建構子實作特徵，而非只是型別而已。但這些都不包含在本 RFC 內。

## 關聯型別建構子的功能

### 宣告與賦值給關聯型別建構子

本 RFC 將提供個非常簡單的語法來定義關聯型別建構子，它會與建立型別建構子的別名用的語法非常相似。使用此語法的目的就是為了避免讓尚未能理解嗨爾卡因斗的使用者使用困難。

```rust
trait StreamingIterator {
   type Item<'a>;
}
```

這樣能清楚表達關聯項目 `Item` 是一個型別建構子，而非只是個型別，因為它有個型別參數附加在旁。

與關聯型別一樣，關聯型別建構子可以被綁定：

```rust
trait Iterable {
    type Item<'a>;
    type Iter<'a>: Iterator<Item = Self::Item<'a>>;
    
    fn iter<'a>(&'a self) -> Self::Iter<'a>;
}
```

此界限套用到了型別建構子的「輸出」，然後參數會被視為高階參數。也就是說上述的界限幾乎等於對特徵加上這樣的界限：

```rust
for<'a> Self::Iter<'a>: Iterator<Item = Self::Item<'a>>
```

在 impl 中賦值給關聯型別建構子的語法與賦值給關聯型別的非常接近：

```rust
impl<T> StreamingIterator for StreamIterMut<T> {
    type Item<'a> = &'a mut [T];
    ...
}
```

### 使用關聯型別建構子建立型別

一旦特徵擁有關聯型別建構子時，它就能套用在任何參數或作用域中的表達項目。這能同時用在特徵的本體內與本體外，使用的語法就與使用關聯型別類似。以下是一些範例：

```rust
trait StreamingIterator {
    type Item<'a>;
    // 在特徵內將生命週期 `'a` 套用至 `Self::Item`。
    fn next<'a>(&'a self) -> Option<Self::Item<'a>>;
}

struct Foo<T: StreamingIterator> {
    // 在特徵外將實際生命週期套用至建構子。
    bar: <T as StreamingIterator>::Item<'static>;
}
```

關聯型別建構子也能用來產生其他型別建構子：

```rust
trait Foo {
    type Bar<'a, 'b>;
}

trait Baz {
    type Quux<'a>;
}

impl<T> Baz for T where T: Foo {
    type Quux<'a> = <T as Foo>::Bar<'a, 'static>;
}
```

最後，關聯型別建構子的生命週期和其他型別建構子的一樣能被省略。加上生命週期省略的話，`StreamingIterator` 的完整定義就可以是：

```rust
trait StreamingIterator {
    type Item<'a>;
    fn next(&mut self) -> Option<Self::Item>;
}
```

### 在界限中使用關聯型別建構子

對於參數的型別是由特徵的關聯型別建構子產生的話，使用者可以用高階特徵界限（Higher-Rank Trait Bounds）來綁定。無論是對型別相等的界限或是對此卡因斗的特徵界限都是有效的：

```rust
fn foo<T: for<'a> StreamingIterator<Item<'a>=&'a [i32]>>(iter: T) { ... }

fn foo<T>(iter: T) where T: StreamingIterator, for<'a> T::Item<'a>: Display { ... }
```

本 RFC 並沒有提出任何允許型別建構子自己綁定的形式，無論是相等界限或特徵界限（當然特徵界限是不可能的）。

## 型別引數的關聯型別建構子

本 RFC 目前所有的範例都專注在生命週期引數的關聯型別建構子。但是本 RFC 一樣有提出導入型別的關聯型別建構子：

```rust
trait Foo {
    type Bar<T>;
}
```

本 RFC **並沒有**打算讓高階特徵界限能進一步接收型別引數，因為這會讓它們更難清楚地表達語義。我們的確有這樣的擴充需求，但這超出本 RFC 的範疇。

型別引數可以透過「Family」模式來達成其他形式的嗨爾卡因斗多型。舉例來說，使用 `PointerFamily` 特徵可以將 Arc 與 Rc 抽象出來：

```rust
trait PointerFamily {
    type Pointer<T>: Deref<Target = T>;
    fn new<T>(value: T) -> Self::Pointer<T>;
}

struct ArcFamily;

impl PointerFamily for ArcFamily {
    type Pointer<T> = Arc<T>;
    fn new<T>(value: T) -> Self::Pointer<T> {
        Arc::new(value)
    }
}

struct RcFamily;

impl PointerFamily for RcFamily {
    type Pointer<T> = Rc<T>;
    fn new<T>(value: T) -> Self::Pointer<T> {
        Rc::new(value)
    }
}

struct Foo<P: PointerFamily> {
    bar: P::Pointer<String>,
}
```

## 特徵界限與 where

### 關聯型別建構子的界限

關聯型別建構子的界限會被視為特徵自己的高階界限。這讓它們的行為能與一般關聯型別的界限一致。舉例來說：

```rust
trait Foo {
    type Assoc<'a>: Trait<'a>;
}
```

就等同於：

```rust
trait Foo where for<'a> Self::Assoc<'a>: Trait<'a> {
    type Assoc<'a>;
}
```

### 關聯型別上的 `where`

另一方面，在關聯型別上使用 where 則會要求每次使用關聯型別時都得滿足其限制。舉例來說：

```rust
trait Foo {
    type Assoc where Self: Sized;
}
```

每次呼叫 `<T as Foo>::Assoc` 時都需要證明 `T: Sized`，就像其他場合需要證明有符合 impl 的界限一樣。

(@nikomatsakis believes that where clauses will be needed on associated type
constructors specifically to handle lifetime well formedness in some cases.
The exact details are left out of this RFC because they will emerge more fully
during implementation.)

## Benefits of implementing only this feature before other higher-kinded polymorphisms

This feature is not full-blown higher-kinded polymorphism, and does not allow
for the forms of abstraction that are so popular in Haskell, but it does
provide most of the unique-to-Rust use cases for higher-kinded polymorphism,
such as streaming iterators and collection traits. It is probably also the
most accessible feature for most users, being somewhat easy to understand
intuitively without understanding higher-kindedness.

This feature has several tricky implementation challenges, but avoids all of
these features that other kinds of higher-kinded polymorphism require:

* Defining higher-kinded traits
* Implementing higher-kinded traits for type operators
* Higher order type operators
* Type operator parameters bound by higher-kinded traits
* Type operator parameters applied to a given type or type parameter

## Advantages of proposed syntax

The advantage of the proposed syntax is that it leverages syntax that already
exists. Type constructors can already be aliased in Rust using the same syntax
that this used, and while type aliases play no polymorphic role in type
resolution, to users they seem very similar to associated types. A goal of this
syntax is that many users will be able to use types which have assocaited type
constructors without even being aware that this has something to do with a type
system feature called higher-kindedness.

# How We Teach This
[how-we-teach-this]: #how-we-teach-this

This RFC uses the terminology "associated type constructor," which has become
the standard way to talk about this feature in the Rust community. This is not
a very accessible framing of this concept; in particular the term "type
constructor" is an obscure piece of jargon from type theory which most users
cannot be expected to be familiar with.

Upon accepting this RFC, we should begin (with haste) refering to this concept
as simply "generic associated types." Today, associated types cannot be
generic; after this RFC, this will be possible. Rather than teaching this as
a separate feature, it will be taught as an advanced use case for associated
types.

Patterns like "family traits" should also be taught in some way, possible in
the book or possibly just through supplemental forms of documentation like
blog posts.

This will also likely increase the frequency with which users have to employ
higher rank trait bounds; we will want to put additional effort into teaching
and making teachable HRTBs.

# Drawbacks
[drawbacks]: #drawbacks

## Adding language complexity

This would add a somewhat complex feature to the language, being able to
polymorphically resolve type constructors, and requires several extensions to
the type system which make the implementation more complicated.

Additionally, though the syntax is designed to make this feature easy to learn,
it also makes it more plausible that a user may accidentally use it when they
mean something else, similar to the confusion between `impl .. for Trait` and
`impl<T> .. for T where T: Trait`. For example:

```rust
// The user means this
trait Foo<'a> {
    type Bar: 'a;
}

// But they write this
trait Foo<'a> {
    type Bar<'a>;
}
```

## Not full "higher-kinded types"

This does not add all of the features people want when they talk about higher-
kinded types. For example, it does not enable traits like `Monad`. Some people
may prefer to implement all of these features together at once. However, this
feature is forward compatible with other kinds of higher-kinded polymorphism,
and doesn't preclude implementing them in any way. In fact, it paves the way
by solving some implementation details that will impact other kinds of higher-
kindedness as well, such as partial application.

## Syntax isn't like other forms of higher-kinded polymorphism

Though the proposed syntax is very similar to the syntax for associated types
and type aliases, it is probably not possible for other forms of higher-kinded
polymorphism to use a syntax along the same lines. For this reason, the syntax
used to define an associated type constructor will probably be very different
from the syntax used to e.g. implement a trait for a type constructor.

However, the syntax used for these other forms of higher-kinded polymorphism
will depend on exactly what features they enable. It would be hard to design
a syntax which is consistent with unknown features.

# Alternatives
[alternatives]: #alternatives

## Push HRTBs harder without associated type constructors

An alternative is to push harder on HRTBs, possibly introducing some elision
that would make them easier to use.

Currently, an approximation of `StreamingIterator` can be defined like this:

```rust
trait StreamingIterator<'a> {
   type Item: 'a;
   fn next(&'a self) -> Option<Self::Item>;
}
```

You can then bound types as `T: for<'a> StreamingIterator<'a>` to avoid the
lifetime parameter infecting everything `StreamingIterator` appears.

However, this only partially prevents the infectiveness of `StreamingIterator`,
only allows for some of the types that associated type constructors can
express, and is in generally a hacky attempt to work around the limitation
rather than an equivalent alternative.

## Impose restrictions on ATCs

What is often called "full higher kinded polymorphism" is allowing the use of
type constructors as input parameters to other type constructors - higher order
type constructors, in other words. Without any restrictions, multiparameter
higher order type constructors present serious problems for type inference.

For example, if you are attempting to infer types, and you know you have a
constructor of the form `type, type -> Result<(), io::Error>`, without any
restrictions it is difficult to determine if this constructor is
`(), io::Error -> Result<(), io::Error>` or `io::Error, () -> Result<(), io::Error>`.

Because of this, languages with first class higher kinded polymorphism tend to
impose restrictions on these higher kinded terms, such as Haskell's currying
rules.

If Rust were to adopt higher order type constructors, it would need to impose
similar restrictions on the kinds of type constructors they can receive. But
associated type constructors, being a kind of alias, inherently mask the actual
structure of the concrete type constructor. In other words, if we want to be
able to use ATCs as arguments to higher order type constructors, we would need
to impose those restrictions on *all* ATCs.

We have a list of restrictions we believe are necessary and sufficient; more
background can be found in [this blog post](http://smallcultfollowing.com/babysteps/blog/2016/11/09/associated-type-constructors-part-4-unifying-atc-and-hkt/)
by nmatsakis:

* Each argument to the ATC must be applied
* They must be applied in the same order they appear in the ATC
* They must be applied exactly once
* They must be the left-most arguments of the constructor

These restrictions are quite constrictive; there are several applications of
ATCs that we already know about that would be frustrated by this, such as the
definition of `Iterable` for `HashMap` (for which the item `(&'a K, &'a V)`,
applying the lifetime twice).

For this reason we have decided **not** to apply these restrictions to all
ATCs. This will mean that if higher order type constructors are ever added to
the language, they will not be able to take an abstract ATC as an argument.
However, this can be maneuvered around using newtypes which do meet the
restrictions, for example:

```rust
struct IterItem<'a, I: Iterable>(I::Item<'a>);
```

# Unresolved questions
[unresolved]: #unresolved-questions
