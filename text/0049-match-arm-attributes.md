- Start Date: 2014-03-20
- RFC PR: [rust-lang/rfcs#49](https://github.com/rust-lang/rfcs/pull/49)
- Rust Issue: [rust-lang/rust#12812](https://github.com/rust-lang/rust/issues/12812)
- Translators: [[@FizzyElt](https://github.com/FizzyElt)]
- Commit: [The commit link this page based on](https://github.com/rust-lang/rfcs/blob/master/text/0049-match-arm-attributes.md)
- Updated: 2022-05-17

# 概要

允許屬性在匹配分支上

# 動機

有時希望使用屬性來注釋匹配語句的分支，例如條件編譯 `#[cfg]` 或是分支權重（後者為最重要的用途）。


對於條件編譯，解決辦法是使用 `#[cfg]` 來重複包含函式的全部部分。一個案例研究是 [sfackler's bindings to OpenSSL](https://github.com/sfackler/rust-openssl)，在多數的發行版本中移除了 SSLv2 支持，因此 Rust bindings 部分需要被條件禁用。支持各種不同 SSL 版本的明顯方法是一個枚舉。

```rust
pub enum SslMethod {
    #[cfg(sslv2)]
    /// Only support the SSLv2 protocol
    Sslv2,
    /// Only support the SSLv3 protocol
    Sslv3,
    /// Only support the TLSv1 protocol
    Tlsv1,
    /// Support the SSLv2, SSLv3 and TLSv1 protocols
    Sslv23,
}
```

然而，所有的 `match` 只能在 `cfg` 是有效的時候提起 `Sslv2`，例如下面內容是無效的：

```rust
fn name(method: SslMethod) -> &'static str {
    match method {
        Sslv2 => "SSLv2",
        Sslv3 => "SSLv3",
        _ => "..."
    }
}
```

一個有效的方法有兩個定義：`#[cfg(sslv2)] fn
name(...)` 和 `#[cfg(not(sslv2)] fn name(...)`
。前者有 `Sslv2` 的分支，後者沒有。顯然，對於枚舉中每個額外的 `cfg` 變體，這都會以指數型的方式爆炸。

分支權重將允許仔細的微優化器（micro-optimiser）通知編譯器，例如，鮮少採取某個匹配分支：

```rust
match foo {
    Common => {}
    #[cold]
    Rare => {}
}
```

# 詳細設計

一般的屬性語法，應用於整個匹配分支。

```rust
match x {
    #[attr]
    Thing => {}

    #[attr]
    Foo | Bar => {}

    #[attr]
    _ => {}
}
```

# 替代方案

實際上沒有通用的替代方案; 人們也許可以用一些巨集（macro）和輔助函數來解決條件枚舉變體的匹配問題; 但一般來說，這起不了任何作用。

# 未解決的問題

無