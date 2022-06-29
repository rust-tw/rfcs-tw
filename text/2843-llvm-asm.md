- Feature Name: `llvm_asm`
- Start Date: 2019-12-31
- RFC PR: [rust-lang/rfcs#2843](https://github.com/rust-lang/rfcs/pull/2843)
- Rust Issue: [rust-lang/rust#70173](https://github.com/rust-lang/rust/issues/70173)
- Translators: [[@TSOTSI1](https://github.com/TSOTSI1)]
- Commit: [The commit link this page based on](https://github.com/Amanieu/rfcs/blob/9911615f81471229684b0ab704409c9fcae55ede/text/2843-llvm-asm.md)
- Updated: 2022-06-29

# 總結
[總結]: #總結

棄用現有的 `asm!` 巨集（macro），並提供一個名為 `llvm_asm!` 的相同巨集（macro）。功能開關也從 `asm` 重新命名為 `llvm_asm` 。
與 `asm!` 不同的是， `llvm_asm!` 不打算成為穩定版。

# 動機
[動機]: #動機

這個變更將 `asm!` 巨集（macro）釋放了，使它可用於內嵌組合語言項目組新設計的 `asm!` 巨集（macro），
同時為現在使用 `asm!` 的用戶提供一個簡單的方法來保持代碼的正常運作。

對尚未支援新的 `asm!` 巨集（macro）的架構，在（nightly 版本）上執行內嵌組合語言可能會有用。

# 教學式解說
[教學式解說]: #教學式解說

Rust 團隊目前正在重新設計 `asm!` 巨集（macro）。您應該將所有使用 `asm!` 的代碼都替換為 `llvm_asm!` ，以避免代碼在實施新的 `asm!` 巨集（macro）時毀壞。

# 技術文件式解說
[技術文件式解說]: #技術文件式解說

編譯器內所有對 `asm!` 的參考都將改參考 `llvm_asm!` 。
`asm!` 將變成一個單純（已棄用）的 `macro_rules!` ，它會重新導到`llvm_asm!`。
棄用警告將告知使用者 `asm!` 將來的語義會作改變，並邀請他們使用 `llvm_asm!` 來替代。`llvm_asm!` 巨集（macro）將由 `llvm_asm` 功能開關來把控。

# 缺點
[缺點]: #缺點

此變更可能需要人為變更兩次代碼：首先變更為 `llvm_asm!` ，然後再實施新的 `asm!` 巨集（macro）。

# 原理及替代方案
[原理及替代方案]: #原理及替代方案

我們可以跳過棄用期，並同時執行重新命名新的 `asm!` 巨集（macro）。
總之用 Rust（nightly 版本）保證能一次破壞大量代碼，就無需任何過渡期。

# 先驅技術
[先驅技術]: #先驅技術

D 語言也支援兩種形式的內嵌組合語言。[first one][d-asm] 提供用於內嵌組合語言的嵌入式 DSL，它可以在不用 Clobber 情況下直接存取範圍內的變量，但只能在x86和x86_64的架構上使用。
[second one][d-llvm-asm] 是 LLVM 內部內嵌組合語言句法的 RAM 接口，但它只適用於 DSL 的後端架構。

[d-asm]: https://dlang.org/spec/iasm.html
[d-llvm-asm]: https://wiki.dlang.org/LDC_inline_assembly_expressions

# 未解決問題
[未解決問題]: #未解決問題

無

# 未來展望
[未來展望]: #未來展望

當下執行的會在 [new `asm!` macro][inline-asm-rfc] 被執行後替換掉，這會破壞那些尚未轉換 `llvm_asm!` 的代碼。
由於運算元分隔符將從 `:` 更改為 `,` ，所以不會有靜默的錯誤編譯，新的 `asm!` 巨集（macro）會出現語法錯誤，來保證現有 `asm!` 的任何呼叫都會失敗，

[inline-asm-rfc]: https://github.com/rust-lang/rfcs/pull/2873
