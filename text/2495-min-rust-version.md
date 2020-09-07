- Feature Name: `min_rust_version`
- Start Date: 2018-06-28
- RFC PR: [rust-lang/rfcs#2495](https://github.com/rust-lang/rfcs/pull/2495)
- Rust Issue: [rust-lang/rust#65262](https://github.com/rust-lang/rust/issues/65262)
- Translators: [[@weihanglo](https://github.com/weihanglo)]
- Commit: [The commit link this page based on](https://github.com/rust-lang/rfcs/blob/3071138d4ed510d6dfc1f8e1d7e9d4b099ea12e8/text/2495-min-rust-version.md)
- Updated: 2020-09-07

# 總結
[總結]: #總結

在 `Cargo.toml` 的 package 區塊加入 `rust` 欄位，用於指定 crate 的最低支援 Rust 版本（Minimum Supported Rust Version，MSRV）。

```toml
[package]
name = "foo"
version = "0.1.0"
rust = "1.30"
```

# 動機
[動機]: #動機

當前 crate 無任何正式方法指定 MSRV，導致使用者無法在不建構該 crate 的情況下得知 crate 是否可透過他們的工具鏈（toolchain）建構。這也帶來有關如何在提升 MSRV 時管理 crate 版本的爭論，保守的作法是將之視為破壞性改動，這種作法會阻礙整個生態採納新功能，或使得版本號通膨式提升，進而讓下游的 crate 很難跟上。另一方面，若作法稍微寬鬆些，則導致採用較舊編譯器版本的使用者無法成功編譯它們的 crate。

# 教學式解說
[教學式解說]: #教學式解說

若你想指定一個特定 MSRV 版本，請在 `Cargo.toml` 的 `[package]` 區塊中的 `rust` 欄位設定指定的 Rust 版本。如果你建構一個 crate 時有任一依賴要求的 MSRV 比你當前工具鏈更高，會導致編譯錯誤，指明該依賴與它的 MSRV。這個行為可以透過 `--no-msrv-check` 選項來停用。

# 技術文件式解說
[技術文件式解說]: #技術文件式解說

在建構過程（包含 `run`、`test`、`benchmark`、`verify` 與 `publish` 子指令），`cargo` 會以依賴樹（dependency tree）的形式，檢查所有將要建構或檢查的所有 crate 之 MSRV 要求。在依賴樹內但不會被構建的 crate 不會執行此項檢查（例如特定目標平台或可選的 crate）。

`rust` 欄位應至少遵循下列要求：

- 其值需遵守語意化版號且不能有範圍運算子。注意，「1.50」是合法的值，代表「1.50.0」。
- 版本不能比當前 stable 工具鏈高（上傳 crate 時 crates.io 會檢查）。
- 版本不能低於 1.27（此版本將 `package.rust` 欄位從錯誤改為警告）。
- 版本不能比選用的 edition 釋出的版本低，舉例來說，同時出現 `rust = "1.27"` 與 `edition = 2018` 是非法的。

# 未來展望與延伸
[未來展望與延伸]: #未來展望與延伸

## 對版本解析之影響

`rust` 欄位之值（手動設定或 `cargo` 自動選擇）會用於選擇合適的依賴版本。

舉例來說，想像你的 crate 相依 crate `foo`，一個發佈了從 `0.1.0` 到 `0.1.9` 十個版本的 crate，其中 `0.1.0` 到 `0.1.5` 版在 crates.io 上 `Cargo.toml` 的 `rust` 欄位為「1.30」，其他版本則為「1.40」。現在，你建構一個專案用了例如 Rust 1.33 版，`cargo` 會選用 `foo v0.1.5`。`foo v0.1.9` 只會在你用 Rust 1.40 或更高版本建構專案時選用。倘若你嘗試使用 Rust 1.29 建構專案，cargo 會回報錯誤。

`rust` 欄位值也會被檢核。在 crate 建構過程，`cargo` 將檢查所有上游相依是否可以在指定 MSRV 下建構。（例如，檢查給定的 crate 與 Rust 版本限制條件下是否存在解）已除去（yank）的 crate 會略過這個檢查步驟。

期望實作這項功能得以替長久以來對 MSRV 提升是否為破壞性改動的爭論劃下休止符，並讓 crate 作者提升 crate 的 MSRV 不再如此綁手綁腳。（儘管對於已 1.0 的 crate 來說，透過提升修訂號（patch version）來提升 MSRV 次版號，以接納修復嚴重問題的 backport，可能是個有用的慣例）

注意，上述 MSRV 限制與依賴版本解析檢查，可以透過 `--no-msrv-check` 選項停用。

## 發佈時檢查 MSRV

`cargo publish` 將檢查上傳是以 `rust` 欄位指定的工具鏈版本完成，若工具鏈版本有異，`cargo` 會拒絕上傳該 crate。此確保機制避免因為未預期的 MSRV 提升導致錯誤的 `rust` 欄位值。這項檢查可透過既有的 `--no-verify` 選項停用。

## 將 `rust` 欄位設為必填

未來（可能是下一個 edition），我們可以設定新上傳的 crate 的 `rust` 欄位為必填。既有 crate 的 MSRV 會透過 `edition` 決定。換句話說 `edition = 2018` 之 MSRV 必然是 `rust = "1.31"`，而 `edition = "2015"` 則是 `rust = "1.0"`。

`cargo init` 將會使用當前工具鏈使用的版本。

## 基於 `cfg` 的 MSRV

部分 crate 會根據不同目標架構平台或啟用的功能而有不同的 MSRV。可透過以下方式有效指定 MSRV 如何依賴這些配置：

```toml
[package]
rust = "1.30"

[target.x86_64-pc-windows-gnu.package]
rust = "1.35"

[target.'cfg(feature = "foo")'.package]
rust = "1.33"
```

在 `target` 區塊中所有 `rust` 值應等於或高於在 `package` 區塊的 `rust` 值。

若 `target` 的條件為真，`cargo` 會取用該區塊的 `rust` 值。若多個 target 區塊的條件為真，則取用最大值。

## Nightly 與 stable 版本

部分 crate 可能偏好在最新 stable 或 nighly 工具鏈，除了指定版本之外，我們可允許宣告 `stable` 或 `nightly` 值，讓維護者不需追蹤該 crate 的 MSRV 。

對於某些超前沿的 crate（例如： `rocket`）常常因為 Nightly 更新就壞，將可指定特定可成功建構的 Nightly 版本。透過下列語法來達成：

- 自動選擇：`nightly` 此寫法與寫 `stable` 的行為一致，將使用等於當前或更高的 nightly 版本。
- 單一版本：`nightly: 2018-01-01` （主要寫法）
- 列舉：`nightly: 2018-01-01, 2018-01-15`
- 類語意化版本條件：`nightly: >=2018-01-01`、`nightly: >=2018-01-01, <=2018-01-15`、`nightly: >=2018-01-01, <=2018-01-15, 2018-01-20`。（後者會解讀為 「(version >= 2018-01-01 && version <= 2018-01-20) || version == 2018-01-20」）

這些條件或許很嚴苛，盼使用這功能的 crate 一隻手數得出來。

# 缺點
[缺點]: #缺點

- 即使宣告了 MSRV 且檢查過，並無法保持 crate 能夠正確在指定 MSRV 下正確執行，只有合理配置的 CI 能做到此事。
- 更複雜的依賴版本解析演算法。
- 使用 `cargo publish` 配合 MSRV `rust = "stable"` 恐過於保守。

# 替代方案
[替代方案]: #替代方案

- 自動計算 MSRV。
- 不做任何事，依靠 [LTS 發行](https://github.com/rust-lang/rfcs/pull/2483) 的 crate MSRV 提升。
- 允許在 [RFC 2523](https://github.com/rust-lang/rfcs/pull/2523) 中提出基於版本與路徑的 `cfg` 屬性（attribute）

# 先前技術
[先前技術]: #先前技術

早先的提案：

- [RFC 1707](https://github.com/rust-lang/rfcs/pull/1707)
- [RFC 1709](https://github.com/rust-lang/rfcs/pull/1709)
- [RFC 1953](https://github.com/rust-lang/rfcs/pull/1953)
- [RFC 2182](https://github.com/rust-lang/rfcs/pull/2182)（這個非常有爭議的離題了）

# 未解決問題
[未解決問題]: #未解決問題

- 瑣碎的命名問題：`rust` 或 `rustc` 還是 `min-rust-version`
- 額外的檢查？
- 更優質地說明版本解析演算法
- nightly 版本如何與「基於 cfg 的 MSRV」運作？
