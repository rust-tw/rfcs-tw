- Start Date: 2014-09-11
- RFC PR #: [rust-lang/rfcs#198](https://github.com/rust-lang/rfcs/pull/198)
- Rust Issue #: [rust-lang/rust#17177](https://github.com/rust-lang/rust/issues/17177)
- Translators: [[@FizzyElt](https://github.com/FizzyElt)]
- Commit: [The commit link this page based on](https://github.com/rust-lang/rfcs/blob/4009b546172c558a1cfa0f39dd81c896312f73d5/text/0198-slice-notation.md)
- Updated: 2022-05-27

# 概要

此 RFC 增加了*多載切片符號*：

- `foo[]` 對應 `foo.as_slice()`
- `foo[n..m]` 對應 `foo.slice(n, m)`
- `foo[n..]` 對應 `foo.slice_from(n)`
- `foo[..m]` 對應 `foo.slice_to(m)`
- 以上所有的 `mut` 變體

透過`Slice` 跟 `SliceMut` 兩個新特徵（trait）。

它還將 範圍`配對`模式的符號改為 `...`，以表示它們是包含的，而 `..` 在切片中是不包含的。

# 動機

引入此功能有兩個主要動機。

### 符合人體工學

在處理 vector 或其他類型的容器時，切片操作（特別是 `as_slice`）是相當常見的基本操作。我們已經有了透過 `Index` 特徵進行索引的語法，此 RFC 本質上是這特徵的延續。

`as_slice` 運算子尤其重要，自從我們已經擺脫以強迫的方式自動切片，明確地呼叫 `as_slice` 變得非常普遍，這也是語言中[人體工學優先/第一印象](https://github.com/rust-lang/rust/issues/14983)的問題之一。還有一些其他方法可以解決這個特定問題，但這些替代方法具有下方討論的一些缺點（參閱「替代方案」）。

### 錯誤處理協定

我們正在逐漸走向一個類似 Python 的世界，像是當 `n` 超出範圍時，`foo[n]` 會呼叫 `fail!`，而相應的方法像是 `get` 則返回 `Option` 而不是失敗。透過為切片提供類似的符號，我們打開了在整個 vector-like APIs 中遵循相同協定的大門。

# 詳細設計

此設計是對 `Index` 特徵設計的直接延續。我們引入了兩個新特徵（trait），用於不可變與可變切片：

```rust
trait Slice<Idx, S> {
    fn as_slice<'a>(&'a self) -> &'a S;
    fn slice_from(&'a self, from: Idx) -> &'a S;
    fn slice_to(&'a self, to: Idx) -> &'a S;
    fn slice(&'a self, from: Idx, to: Idx) -> &'a S;
}

trait SliceMut<Idx, S> {
    fn as_mut_slice<'a>(&'a mut self) -> &'a mut S;
    fn slice_from_mut(&'a mut self, from: Idx) -> &'a mut S;
    fn slice_to_mut(&'a mut self, to: Idx) -> &'a mut S;
    fn slice_mut(&'a mut self, from: Idx, to: Idx) -> &'a mut S;
}
```

（請注意，此處的可變名稱是命名規則可能更改的一部分，將在單獨的 RFC 中描述）。

這些特徵將在解釋以下符號時使用：

*不可變切片*

- `foo[]` 對應 `foo.as_slice()`
- `foo[n..m]` 對應 `foo.slice(n, m)`
- `foo[n..]` 對應 `foo.slice_from(n)`
- `foo[..m]` 對應 `foo.slice_to(m)`

*可變切片*

- `foo[mut]` 對應 `foo.as_mut_slice()`
- `foo[mut n..m]` 對應 `foo.slice_mut(n, m)`
- `foo[mut n..]` 對應 `foo.slice_from_mut(n)`
- `foo[mut ..m]` 對應 `foo.slice_to_mut(m)`

像 `Index` 一樣，這種表示法的使用將自動取值，就如同對應的方法調用一樣。因此，如果 `T` 實現了 `Slice<uint, [U]>` 和 `s: Smaht<T>`，那麼 `s[]` 就可以編譯並具有 `&[U]` 類型。

請注意，切片是「不包含」（因此 `[n..m]` 是等於 `n <= x < m`），而模式配對中的 `..` 是「包含」。為了避免混淆，我們建議將配對符號更改為 `...` 來做出區別。更改符號而不是解釋的原因是，「不包含」（分別「包含」）解釋是切片（相較於「配對」）的正確默認值。

## 使用此語法的基本理由

用於切片的方括號的選擇很簡單：它與我們的索引表示法契合，並且切片和索引密切相關。

其他一些語言（如 Python 和 Go 和 Fortran）在切片表示法中使用 `:` 而不是 `..`。在 Rust 中，`..` 的選擇受到 Rust 本身其他地方的使用影響，例如固定長度陣列類型 `[T, ..n]`。用於切片的 `..` 在 Perl 和 D 中有先例。

有關程式語言中切片表示法歷史的更多資訊，請參閱 [維基百科](http://en.wikipedia.org/wiki/Array_slicing)。

### `mut` 限定符

可能令人驚訝的是，`mut` 在建議的切片表示法中作為限定符，而不是用於索引表示法。原因是索引包含隱式取消引用。假如 `v: Vec<Foo>` 則 `v[n]` 的類型為 `Foo`，而不是 `&Foo` 或 `&mut Foo`。因此，如果你想透過索引取得可變引用，你可以寫成 `&mut v[n]`。更廣泛地說，這允許我們在解決可變性之前進行解析/類型檢查。

這種對 `Index` 的處理符合 C 的傳統，並允許我們寫成 `v[0] = foo`，而不是 `*v[0] = foo`。

另一方面，這種方法對於切片是有問題的，因為一般來說，他會產生一個 unsized 類型（在 DST 中），當然，切片是為了給你一個指向切片大小的胖指標，我們不希望立即取消引用。但是，這樣的結果是，我們需要預先知道切片的可變性，因為它決定了表達式的類型。

# 缺點

主要的缺點是增加了語言語法的複雜度。這看起來微不足道，尤其是因為這裡的符號本質上是 "完成" 以 `Index` 特徵開始的內容。

## 設計上的限制

與 `Index` 特徵一樣，這迫使結果透過 `＆` 成為一個引用，這可能會排除切片的普遍化。

解決此問題的一個方法是切片方法使用 `self` （依值) 而不是 `&self`，反過來在 `&T` 而不是 `T` 上實現特徵。這個方法是否長期可行將取決於方法解析和自動引用的最終規則。

一般來說，當特徵可以應用於類型 `T` 而不是借用類型 `&T` 時，特徵系統運作的最好。最終，如果 Rust 獲得了更高的類型（HKT），我們可以將特徵中的切片類型 `S` 更改為更高的類型，這樣他就是一個由生命週期索引的類型*家族*。然後，我們可以用 `S<'a>` 來代替返回值中的 `&'a S`。在未來，我們應該可以從當前的 `Index` 和 `Slice` 特徵設計轉換到一個 HKT 版本，而不會破壞相後的兼容性，方法是對實現舊特徵的類型使用新特徵的全面實現（例如 `IndexHKT`）。

# 替代方案

對於改善 `as_slice` 的易用性，有兩個主要替代方案。

## 強制: 自動切片

一種可能性是重新引入某種自動切片的強制。我們曾經有一個從（用現今的話來說）`Vec<T>` 到 `&[T]` 的強制轉換。由於我們不再強制擁有借來的值，我們現在可能想要一個強制 `&Vec<T>` 到 `&[T]` 的轉換：  

```rust
fn use_slice(t: &[u8]) { ... }

let v = vec!(0u8, 1, 2);
use_slice(&v)           // automatically coerce here
use_slice(v.as_slice()) // equivalent
```

不幸的是，添加這樣的強制需要在以下選項中進行選擇：

* 將強制與 `Vec` 和 `String` 連繫起來，這將重新引入對這些原本純粹的函式庫類型的特殊處理，並且意味著其他支援切片的函數庫類型不會受益（違背了一些 DST 的目的）。

* 透過特徵使強制可擴展。然而這是在打開潘朵拉的盒子：這個機制很可能被用來在強制的過程中運行任意的程式碼，所以任何呼叫 `foo(a, b, c)` 都可能涉及運行程式碼來預處理每個參數。雖然我們最終可能想要這種使用者可擴展的強制機制，但在理解程式碼時，這是一個很**大**的步驟，有很多潛在的不利因素，所以我們應該先追求更保守的解決方案。

## 解引用

另一種可能是讓 `String` 實現 `Deref<str>` 和 `Vec<T>` 實現 `Deref<[T]>` 一旦 DST 登陸。這樣做將允許顯式的強制機制，例如：

```rust
fn use_slice(t: &[u8]) { ... }

let v = vec!(0u8, 1, 2);
use_slice(&*v)          // take advantage of deref
use_slice(v.as_slice()) // equivalent
```

但是，這樣做至少有兩個缺點：

* 目前不清楚方法解析規則最終將如何與 `Deref` 互動。特別是，一個領先的提議是，對於一個智慧指標 `s: Smaht<T>` ，當你呼叫 `s.m(...)` 時，對於 `Smaht<T>` 只有 *內置* 方法 `m` 被考慮到；僅考慮最大引用值 `*s` 的特徵方法。

  使用這樣的解析策略，為 `Vec` 實現 `Deref` 將使得我們無法在 `Vec` 類型上使用特徵方法 ，除非透過 UFCS，這嚴重限制了程式設計師為 `Vec` 有效地實現新特徵的能力。

* 將 `Vec` 作為一個圍繞切片的智慧指標的想法，以及如上所述的 `&*v` 的使用，有點違反直覺，特別是對於這樣的基本類型。

追根究底，無論如何，切片的符號本身似乎是可取的，如果它能消除對 `Vec` 和 `String` 實現 `Deref` 的需要，那就更好了。