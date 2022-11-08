- Feature Name: generic_associated_types
- Start Date: 2016-04-29
- RFC PR: [rust-lang/rfcs#1598](https://github.com/rust-lang/rfcs/pull/1598)
- Rust Issue: [rust-lang/rust#44265](https://github.com/rust-lang/rust/issues/44265)
- Translators: [@wusyong](https://github.com/wusyong)
- Commit: [a7cd910](https://github.com/rust-lang/rfcs/commit/a7cd91048eea3d7ae83bec20446e62bad0c45381)
- Updated: 2022-11-02

# 總結
[summary]: #總結

讓型別建構子（type constructor）能與特徵（trait）相互關聯。在 Rust 使用者最想要的功能中，有個通常稱為「高階種類型別（higher-kinded types）」的功能，而這是達成這項目標的其中一步。關聯型別建構子（associated type constructors）這項特定的功能可以解決高階種類其中一項最普遍的使用情境，比起其他形式的高階種類多型（polymorphism）更容易擴充至原有的型別系統，而且能兼容未來可能所導入的更複雜形式的高階種類多型。

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

這樣的特徵無法用現在的 Rust 表現出來，因為它需要依賴某種形式的高階種類多型。本 RFC 將擴充 Rust 來確保此特定形式的高階種類多型，也就是我們在此所稱作的關聯型別建構子。此功能有各種應用場景，但最主要的應用都還是與 `StreamingIterator` 類似：定義的特徵所持有的型別能附有一個生命週期，並與接收型別所借用的生命週期相互關聯。

# 設計細節
[design]: #設計細節

## 背景：什麼是種類（Kind）？

「高階種類型別」算是比較籠統的詞彙，容易與其他語言功能混淆，而造成我們想表達的概念不精確。所以本 RFC 將簡單介紹種類的概念。種類通常被稱為「型別的型別」，但這樣的解釋對於想理解的人幫助並不大，反而對已經了解這些概念的人才說得通。所以讓我們嘗試用型別比喻來了解種類。

在型別完善的語言中，每個表達式（expression）都有個型別。許多表達式都有所謂的「基本型別」，也就是語言中的原生型別而且無法用其他型別表達。在 Rust 中，`bool`、`i64`、`usize` 與 `char` 都是基本型別的明顯例子。另一方面，也就會有型別會是由其他型別組成，函式就是屬於這樣的例子。讓我們看看下面這個簡單的函式：

```rust
fn not(x: bool) -> bool {
   !x
}
```

`not` 的型別為 `bool -> bool`（抱歉這邊開始會使用些與 Rust 不大一樣的語法），而  `not(true)` 的型別則是 `bool`。注意這樣的不同點正是了解高階種類的關鍵。

在種類的分析中，`bool`、`char`、`bool -> bool` 等等型別都有 `type` 這種種類。每個型別都有  `type` 種類。然而 `type` 只是基本種類，就像 `bool` 只是個基本型別，我們能寫出更複雜的種類像是 `type -> type`，這樣的種類其中一個範例就是 `Vec` 能接受一個型別然後產生出一個型別。而 `Vec` 的種類與 `Vec<i32>` 的種類（也就是 `type`）之間的差異，就相對應於 `not` 與 `not(true)` 之間的差異。另外注意 `Vec<T>` 的種類和 `Vec<i32>` 一樣仍然是 `type`。盡管 `T` 是個型別參數，`Vec<T>` 還是接收了一個型別，就像 `not(x)` 的型別仍然是 `bool`，盡管 `x` 是個變數一樣。

Rust 相對不尋常的特色是它有**兩種**基本種類，相較於許多語言處理高階種類時只會有一種種類 `type`。Rust 的另一個基本種類就是生命週期參數。如果你有個型別像是 `Foo<'a>`，`Foo` 的種類就是 `lifetime -> type`。

而高階種類當然也可以接受多重引數，`Result` 的種類就是 `type, type -> type`，而  `vec::Iter<'a, T>` 中的 `vec::Iter` 則會有 `lifetime, type -> type` 這樣的種類。

高階種類的表達通常就稱為「型別運算子（type operators）」，運算出一個型別的型別運算子就稱作「型別建構子（type constructors）」。也有其他型別運算子能運算出其他型別運算子，而且還有更高階的型別運算子能接收型別運算子作為引數，讓它們的種類變成像是 `(type -> type) -> type`。本 RFC 將不會討論這些更進階的範疇。

本 RFC 更明確的目標就是要讓型別建構子可以與特徵相互關聯，就像你現在的特徵能與函式、型別與 consts 相互關聯一樣。型別建構子還有包含其他形式的多型，像是對型別建構子實作特徵，而非只是型別而已。但這些都不包含在本 RFC 內。

## 關聯型別建構子的功能

### 宣告與賦值給關聯型別建構子

本 RFC 將提供個非常簡單的語法來定義關聯型別建構子，它會與建立型別建構子的別名用的語法非常相似。使用此語法的目的就是為了避免讓尚未能理解高階種類的使用者使用困難。

```rust
trait StreamingIterator {
   type Item<'a>;
}
```

這樣能清楚表達關聯項目 `Item` 是一個型別建構子，而非只是個型別，因為它有個型別參數附加在旁。

與關聯型別一樣，關聯型別建構子可以用在界限內：

```rust
trait Iterable {
    type Item<'a>;
    type Iter<'a>: Iterator<Item = Self::Item<'a>>;
    
    fn iter<'a>(&'a self) -> Self::Iter<'a>;
}
```

此界限套用到型別建構子的「輸出」，然後參數會被視為高階參數。也就是說上述的界限幾乎等於對特徵加上這樣的界限：

```rust
for<'a> Self::Iter<'a>: Iterator<Item = Self::Item<'a>>
```

在 `impl` 中賦值給關聯型別建構子的語法與賦值給關聯型別的非常接近：

```rust
impl<T> StreamingIterator for StreamIterMut<T> {
    type Item<'a> = &'a mut [T];
    ...
}
```

### 使用關聯型別建構子建立型別

一旦特徵擁有關聯型別建構子時，它就能套用在作用域中的任何參數或表達項目。這能同時用在特徵的本體內與本體外，使用的語法就與使用關聯型別類似。以下是一些範例：

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

對於參數的型別是由特徵的關聯型別建構子產生的話，使用者可以用高階特徵界限（Higher-Rank Trait Bounds）來綁定。無論是對型別相等的界限或是對此種類的特徵界限都是有效的：

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

型別引數可以透過「Family」模式來達成其他形式的高階種類多型。舉例來說，使用 `PointerFamily` 特徵可以將 Arc 與 Rc 抽象出來：

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

(@nikomatsakis 相信在某些場合的 where 會需要關聯型別建構子指明處理生命週期。實際細節將不會包含在本 RFC，因為這在實際實作時才會更清楚。)

## 在其他高階種類多型之前只實作此功能的優點

此功能並非完整的高階種類多型，也無法允許那些在 Haskell 中熱門的抽象形式，但它能提供 Rust 獨一無二的高階種類多型使用情境。像是 `StreamingIterator` 與各式集合的特徵。這很可能也會是許多使用者最需要用到的功能，讓還不知道高階種類的人也能輕易直觀地使用。

此功能會有些麻煩的實作挑戰，但也避免了其他所有高階種類多型所需的功能像是：

* 定義高階種類特徵
* 對型別運算子實作高階種類
* 高階型別運算子
* 高階種類特徵綁定的型別運算子參數
* 型別運算子參數套用至一個給予的型別或型別參數

## 建議語法的優點

建議語法的優點在於它建立在已經存在的語法上，型別建構子已經能在 Rust 中使用相同的語法建立別名。雖然型別別名在型別解析上沒有多型的意義，對於使用者來說他們非常類似於關聯型別。此語法的目標就是要讓許多使用者能夠輕易使用具有關聯型別建構子的型別，就算他們沒有察覺到這和高階種類有關。

# 我們如何教導這個
[how-we-teach-this]: #我們如何教導這個

在本 RFC 我們使用「關聯型別建構子」，這是因為這在 Rust 社群中已經很常拿來使用作為此功能的討論了。但這並沒有很輕易地表達此概念，尤其光是型別理論中的「型別建構子」就已經足以讓人望之卻步了，多數使用者可能並不熟悉這樣的詞彙。

在接受本 RFC 後，我們應該開始稱呼此概念為「泛型關聯型別（generic associated types）」就好，現在的關聯型別還無法使用泛型。在此 RFC 後，這就化為可能了。與其教導這是一個不同的功能，不如介紹說這會是關聯型別的進階使用情境。

像是「Family」特徵的模式也需要同時教導，我們可能會寫進書裡或是某種形式的文件就好，像是網誌文章等。

這也可能會增加使用者使用高階特徵界限的頻率，我們可能會需要挹注更多資源在高階特徵界限上。

# 缺點
[drawbacks]: #缺點

## 增加語言複雜度

這會對語言增加一定的複雜度，型別建構子將能夠多型產生，而且型別系統也需要幾個擴充，這都讓實作更加複雜。

除此之外，雖然語法設計旨在易於學習此功能，但它更有可能產生模糊空間讓使用者意外使用到，而不是他們原本想寫的，這就像 `impl .. for Trait` 和 `impl<T> .. for T where T: Trait` 之間產生的混淆。舉例來說：

```rust
// 使用者其實想寫這樣
trait Foo<'a> {
    type Bar: 'a;
}

// 但他們寫成這樣
trait Foo<'a> {
    type Bar<'a>;
}
```

## 並非完整的「高階種類型別」

本 RFC 並沒有加入當大家講到高階種類型別時的所有功能。舉例來說，它並沒有辦法提供特徵像是 `Monad`。有些人傾向於一次實作完所有這些功能。然而，此功能是能夠兼容於其他形式的高階種類多型的，且並沒有將他們的可能性排除掉。事實上，它還解決了一些會影響其他高階種類形式的實作細節，為它們開拓了可能的路徑，像是 Partial Application。

## 語法與其他形式的高階種類多型都不一樣

雖然建議的語法非常近似於關聯型別與型別別名的語法，其他形式的高階種類多型可能無法使用相同的語法來表達。基於此原因，定義關聯型別建構子的語法可能會與，舉例來說，對特徵實作型別建構子的語法不同。

不過這些其他形式的高階種類多型也將會依到它們實際的功能來決定語法。要設計一個未知功能的語法是非常困難的。

# 替代方案
[alternatives]: #替代方案

## 進一步強化高階特徵界限而不是關聯型別建構子

其中一種替代方案是強化高階特徵界限，有可能是加入一些省略規則，讓它們更容易使用。

目前可能的 `StreamingIterator` 也許能被定義成這樣：

```rust
trait StreamingIterator<'a> {
   type Item: 'a;
   fn next(&'a self) -> Option<Self::Item>;
}
```

這樣你就可以綁定型別成 `T: for<'a> StreamingIterator<'a>` 來避免每次出現 `StreamingIterator` 時都會被生命週期感染。

然而這樣僅避免了 `StreamingIterator` 的傳染性，只能用在一些關聯型別建構子能夠表達的型別，而且這更像是針對限制下的權變措施，而非等價的替代方案。

## 對關聯型別建構子施加限制

我們常稱呼的「完整高階種類多型」能允許使用型別建構子作為其他型別建構子的輸入參數，換句話說就是高階型別建構子。在沒有任何限制的情況下，多重參數的高階型別建構子會對型別介面帶來嚴重的問題。

舉例來說，當你嘗試推導型別時，而且你知道你有個建構子的形式為 `type, type -> Result<(), io::Error>`，如果沒有任何限制的話，我們很難判斷此建構子到底是 `(), io::Error -> Result<(), io::Error>` 還是 `io::Error, () -> Result<(), io::Error>`。

有鑑於此，將高階種類多型視為第一公民的語言通常都會對這些高階種類施加限制，像是 Haskell 的柯里（currying）規則。

如果 Rust 也要採納高階型別建構子的話，我們需要對型別建構子可以接收的種類施加類似的限制。但關聯型別建構子已經是一種別名，其自然就囊括了實際型別建構子的結構。換句話說，如果我們想要使用關聯型別建構子作為高階型別建構子的引數，我們需要將那些限制施加於**所有**的關聯結構建構子。

我們已經有一份我們認為必要且足夠的限制清單，更多細節可以在 nmatsakis 的[網誌文章](http://smallcultfollowing.com/babysteps/blog/2016/11/09/associated-type-constructors-part-4-unifying-atc-and-hkt/)找到：

* 關聯型別建構子的每個引數都必須被套用到
* 它們必須以它們在關聯型別建構子出現的相同順序來套用
* 它們必須套用恰好一次而已
* 它們必須是建構子的最左引數

這些限制已經非常有建設性了，我們已經知道有一些關聯型別建構子的應用會被受限於此，像是 `Iterable` 給 `HashMap` 的定義（項目 `(&'a K, &'a V)` 要套用兩次生命週期）。

有鑑於此，我們決定**不要**對所有關聯型別建構子施加這些限制。這代表如果到時候有高階型別建構子加入語言中的話，它們將無法將關聯型別建構子作為引數。然而，這其實能透過新型別（newtypes）來滿足限制，舉例來說：

```rust
struct IterItem<'a, I: Iterable>(I::Item<'a>);
```

# 未解決問題
[unresolved]: #未解決問題
