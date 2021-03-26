- Feature Name: `const-control-flow`
- Start Date: 2018-01-11
- RFC PR: [rust-lang/rfcs#2342](https://github.com/rust-lang/rfcs/pull/2342)
- Rust Issue: [rust-lang/rust#49146](https://github.com/rust-lang/rust/issues/49146)
- Translators: [[@CYBAI](https://github.com/CYBAI)]
- Commit: [The commit link this page based on](https://github.com/rust-lang/rfcs/blob/dfe697106478a52bddc000477e8cd0621bcc1a20/text/2342-const-control-flow.md)
- Updated: 2020-09-05

# 總結
[總結]: #總結

透過此功能，可以在常數求值（const evaluation）中使用 `if` 及 `match` 並使他們被延遲求值。簡單來說，這項功能允許我們寫 `if x < y { y - x } else { x - y }`；即使當使用非負型別時在 `x < y` 的 `else` 分支會報溢位錯誤 (overflow error)。

# 動機
[動機]: #動機

在常數宣告中使用條件式對於建立像是 `NonZero::new` 的 `const fn` 及 直譯判定（interpreting assertions）來說很重要。

# 教學式解說
[教學式解說]: #教學式解說

如果你寫

```rust
let x: u32 = ...;
let y: u32 = ...;
let a = x - y;
let b = y - x;
if x > y {
    // do something with a
} else {
    // do something with b
}
```

這支程式永遠都會 panic（除非 `x` 和 `y` 同時是 `0`）因為不管是 `x - y` 或是 `y - x` 都會造成溢位。為了解決此問題，我們必須把 `let a` 及 `let b` 個別搬進 `if` 及 `else` 中。

```rust
let x: u32 = ...;
let y: u32 = ...;
if x > y {
    let a = x - y;
    // do something with a
} else {
    let b = y - x;
    // do something with b
}
```

當改用常數時，上面的寫法就會出現新問題：

```rust
const X: u32 = ...;
const Y: u32 = ...;
const FOO: SomeType = if X > Y {
    const A: u32 = X - Y;
    ...
} else {
    const B: u32 = Y - X;
    ...
};
```

`A` 和 `B` 會比 `FOO` 先被求值，因為常數在定義上就是「常數」，所以不應被求值順序影響。這項假設在有錯誤的情況下並不成立，因為錯誤屬於副作用（side effects），因此不純（pure）。

為了解決此問題，我們必須把中介常數消掉並改為直接對 `X - Y` 及 `Y - X` 求值。

```rust
const X: u32 = ...;
const Y: u32 = ...;
const FOO: SomeType = if X > Y {
    let a = X - Y;
    ...
} else {
    let b = Y - X;
    ...
};
```

# 技術文件式解說
[技術文件式解說]: #技術文件式解說

`if` 或是在 variant 沒有欄位的 enums 上做 `match` 會在 HIR -> MIR 階段時，被轉譯成 `switchInt` 終止器（terminator）。Mir 直譯器現在將會針對那些終止器求值（之前就可以了）。

在 variant 沒有欄位的 enums 上做 `match` 會被轉譯成 `switch`，表示他會被檢查 discriminant 或是在 packed enums（例如 `Option<&T>`）的情況會運算 discriminant（這個情況 discriminant 沒有特別的記憶體位址，但他會把所有的零視為 `None`，並把其他的值都當作 `Some`）。當進入 `match` 的分支時，匹配上的值基本上會被 transmute 成 enum 的 variant 型別，如此一來可以允許其他程式碼來存取該 enum 的欄位。

# 缺點
[缺點]: #缺點

這項功能容易造成任意「常數」值（如：`size_of::<T>()` 或是特定的平台常數）編譯失敗。

# 原理及替代方案
[原理及替代方案]: #原理及替代方案

## 利用中介 const fns 來破壞立即常數求值（eager const evaluation）

如果寫成

```rust
const X: u32 = ...;
const Y: u32 = ...;
const AB: u32 = if X > Y {
    X - Y
} else {
    Y - X
};
```

`X - Y` 或是 `Y - X` 其中一方有可能會報錯，這時必須加入中介 const fn

```rust
const X: u32 = ...;
const Y: u32 = ...;
const fn foo(x: u32, y: u32) -> u32 {
    if x > y {
        x - y
    } else {
        y - x
    }
}
const AB: u32 = foo(X, Y);
```

const fn 的 `x` 和 `y` 參數未知，無法做常數求值（const evaluate）。當提供此 const fn 參數並求值時，只會對相應的分支求值。

# 未解決問題
[未解決問題]: #未解決問題
