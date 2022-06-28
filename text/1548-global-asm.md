- Feature Name: global_asm
- Start Date: 2016-03-18
- RFC PR: [rust-lang/rfcs#1548](https://github.com/rust-lang/rfcs/pull/1548)
- Rust Issue: [rust-lang/rust#35119](https://github.com/rust-lang/rust/issues/35119)
- Translators: [[@TSOTSI1](https://github.com/TSOTSI1)]
- Commit: [The commit link this page based on](https://github.com/rust-lang/rfcs/blob/f4b8b61a414298ba0f76d9b786d58ccdc34a44bb/text/1548-global-asm.md)
- Updated: 2022-06-29

# 總結
[總結]: #總結

此 RFC 通過添加 `global_asm!` 巨集（macro）公開了 LLVM 對 [module-level inline assembly](http://llvm.org/docs/LangRef.html#module-level-inline-assembly) 的支持。語法非常簡單：它只需要一個包含組合語言代碼的字符串文字。

範例:
```rust
global_asm!(r#"
.globl my_asm_func
my_asm_func:
    ret
"#);

extern {
    fn my_asm_func();
}
```

# 動機
[動機]: #動機

此功能有兩個主要使用案例。首先，它允許完全在組合語言中寫入函式，這大大消除了對 `naked` 屬性的需要。這主要適用於使用自訂呼叫規則（如中斷處置器）的函式。

另一個重要的使用案例是，它允許外部組合語言檔案在 Rust 模組中使用，而無需駭入編譯系統。

```rust
global_asm!(include_str!("my_asm_file.s"));
```

組合語言檔案也可以由 `build.rs` 預處理或產生（例如使用c預處理器），這將在 cargo 輸出目錄中產生輸出檔案：

```rust
global_asm!(include_str!(concat!(env!("OUT_DIR"), "/preprocessed_asm.s")));
```

# 詳細設計
[詳細設計]: #詳細設計

見上文所述，沒有要多補充的。巨集（macro）會直接映射到 LLVM 的 `module asm`。

# 缺點
[缺點]: #缺點

像`asm!`一樣，這個功能取決於 LLVM 的集成組合語言

# 替代方案
[替代方案]: #替代方案

包含外部組合語言的現有方式是使用 `build.rs` 中的 gcc 編譯組合語言檔案，並將其連結到 Rust 程式中作為靜態庫。

對於完全以組合語言寫入的函式的替代方案是添加一個 [`#[naked]` function attribute](https://github.com/rust-lang/rfcs/pull/1201).

# 未解決問題
[未解決問題]: #未解決問題

無
