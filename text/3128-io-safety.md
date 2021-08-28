- Feature Name: `io_safety`
- Start Date: 2021-05-24
- RFC PR: [rust-lang/rfcs#3128](https://github.com/rust-lang/rfcs/pull/3128)
- Rust Issue: [rust-lang/rust#87074](https://github.com/rust-lang/rust/issues/87074)
- Translators: [[@weihanglo](https://github.com/weihanglo)]
- Commit: [The commit link this page based on](https://github.com/rust-lang/rfcs/blob/88b30e7a0b7ac5b0f52370a58b70c165a9a252af/text/3128-io-safety.md)
- Updated: 2021-08-2o

# 總結
[總結]: #總結

透過引進**輸入輸出安全性**（I/O safety）之概念與一系列新型別與特徵，保障 `AsRaw` 及相關特徵的使用者對原始資源 handle （raw resource handle）之使用，以彌補 Rust 封裝邊界的漏洞。

# 動機
[動機]: #動機

Rust 標準函式庫幾乎算是已經提供了**輸入輸出安全性**，保證程式的一部分若私自持有一個原始 handle （raw handle），其他部分就無法存取。例如 [`FromRawFd::from_raw_fd`] 標示為不安全，它不允許使用者在 safe Rust 之下執行如 `File::from_raw_fd(7)` 等操作，或是在程式各處私自持有檔案描述子（file descriptor）來執行輸入輸出。

不過仍有漏網之魚。許多函式庫的 API 透過接受 [`AsRawFd`]/[`IntoRawFd`] 來執行輸入輸出操作：

```rust
pub fn do_some_io<FD: AsRawFd>(input: &FD) -> io::Result<()> {
    some_syscall(input.as_raw_fd())
}
```

`AsRawFd` 並無限制 `as_raw_fd` 的回傳值，所以 `do_some_io` 最終會對任意 `RawRd` 的值執行輸入輸出。由於 [`RawFd`] 本身實作了 `AsRawFd`，甚至可以寫出 `do_some_io(&7)`。

這會使得程式[存取錯誤的資源]，更甚者為其他地方私有的原始 handle 建立多個別名（alias），從而打破封裝的邊界，造成[遠處來的詭異行為]。

而在特殊情況下，違反輸入輸出安全性恐導致違反記憶體安全性。舉例來說，理論上替透過 Linux [`memfd_create`] 系統呼叫建立出來的檔案描述子打造  `mmap` 的安全封裝，並將之傳給 safe Rust 是可行的，畢竟它就是匿名的被開啟的檔案（anonymous open file），表示其他行程（process）無法存取之。然而，在沒有輸入輸出安全性且沒有永久封閉該檔案的情況下，該程式中其他程式碼恐意外地對該檔案描述子呼叫 `write` 或 `ftruncate`，進而打破記憶體安全性 `&[u8]` 不變的規則。

這個 RFC 透過以下幾點，開闢一條逐步關閉此漏洞的道路：

- 一個新概念：輸入輸出安全性。其概念會撰寫在標準函式庫文件中。
- 一系列全新的型別與特徵。
- 替 [`from_raw_fd`]/[`from_raw_handle`]/[`from_raw_socket`] 撰寫新文件，解釋就輸入輸出安全性而言，它們為何不安全，順便解決出現好[幾][幾][次][次]的相同問題。

[幾]: https://github.com/rust-lang/rust/issues/72175
[次]: https://users.rust-lang.org/t/why-is-fromrawfd-unsafe/39670
[存取錯誤的資源]: https://cwe.mitre.org/data/definitions/910.html
[遠處來的詭異行為]: https://en.wikipedia.org/wiki/Action_at_a_distance_(computer_programming)

# 教學式解說
[教學式解說]: #教學式解說

## 輸入輸出安全性概念

Rust 標準函式庫提供了低階型別，以表示原始的作業系統資源 handle：類 Unix 平台的 [`RawFd`] 和 Windows 上的 [`RawHandle`] 與 [`RawSocket`]。然而，它們並沒有提供任何自身的行為，而是僅作為一個標識符（identifier），並在低階的作業系統 API 間傳遞。

這些原始 handle 可以視為原始指標（raw pointer），具有相同的危險性。雖然**取得**一個原始指標是安全的，但當該原始指標是非法指標，或是比其指向記憶體之處活得更久是，對原始指標**取值**（dereference）可能引發未定義行為（undefined behavior）。無獨有偶，透過 [`AsRawFd::as_raw_fd`] 或類似方法**取得**一個原始 handle 是安全的，但當它並非合法 handle 或在關閉之後才拿來用時，用該 handle 來執行輸入輸出恐導致損毀的輸出結果，或遺失或洩漏輸入資料，或違反封裝的邊界。在這兩個案例中，影響不僅限於本地，也會影響程式的其他部分。保護原始指標免於危險稱作記憶體安全性，所以保護原始 handle 免於危險我們叫它**輸入輸出安全性**。

Rust 標準函式庫也有高階型別如 [`File`] 和 [`TcpStream`] 來提供高階的作業系統 API 界面，這些型別包裝了原始 handle。

這些高階型別同時實作了在類 Unix 平台的 [`FromRawFd`] 特徵，以及 Windows 上的 [`FromRawHandle`]/[`FromRawSocket`] 。這些特徵提供許多函式，會封裝底層的值來產生高階的值。由於這些函式無法確保輸入輸出的安全性，因此標示為不安全。型別系統並不會限制這些型別傳入：

```rust
    use std::fs::File;
    use std::os::unix::io::FromRawFd;

    // 建立一個檔案。
    let file = File::open("data.txt")?;

    // 從任意整數值構建一個 `File`，這個型別能通過檢查，但 7 可能在執行期間無法
    // 被識別為任何活生生的資源，或它可能不慎指向程式其他地方封裝好的原始 handle
    // 一個 `unsafe` 區塊告知呼叫者對此有責任，需使其免於這些風險。
    let forged = unsafe { File::from_raw_fd(7) };

    // 取得一個 `file` 內部的原始 handle 的副本。
    let raw_fd = file.as_raw_fd();

    // 關閉 `file`。
    drop(file);

    // 開啟一些無關的檔案。
    let another = File::open("another.txt")?;

    // 其他對 `raw_fd`（也就是 `file` 內部的原始 handle）的使用有可能使其生命週
    // 期長於與作業系統的關聯。這恐導致意外建立別名指向其他封裝起來的 `File`
    // 實例，例如 `another`。因此，一個 `unsafe` 區塊告知呼叫者對此有責任，需使
    // 其免於這些風險。
    let dangling = unsafe { File::from_raw_fd(raw_fd) };
```

呼叫端必須確保傳入 `from_raw_fd` 的值一定是從作業系統回傳所得，並且 `from_raw_fd` 的回傳值的生命週期不會長於和作業系統關聯的 handle。

雖然將輸入輸出安全性作為明確的概念是個新作法，但其實它也反映出許多常見的實踐。除了引入一些新型別與特徵及其實作外，Rust `std` 並不需改變已經穩定的界面。在推行之初，並不需讓整個 Rust 生態系馬上支援輸入輸出安全性，而是可以漸進式地採用之。

## `OwnedFd` 與 `BorrowedFd<'fd>`

這兩個型別概念上將取代 `RawFd`，並分別表示擁有和借用的 handle 值。`OwnedFd` 擁有一個檔案描述子，當 `OwnedFd` drop 時就會關閉其檔案描述子。`BorrowedFd` 的生命週期標示該檔案描述子被借用多久。這些型別皆會自動實施其輸入輸出安全性不變的規則。

至於在 Windows 上，會以 `Handle` 和 `Socket` 形式呈現相應的型別。

這些型別在輸入輸出扮演的角色可類比 Rust 既有的記憶體管理相關型別：

| 型別             | 類似於       |
| ---------------- | ------------ |
| `OwnedFd`        | `Box<_>`     |
| `BorrowedFd<'a>` | `&'a _`      |
| `RawFd`          | `*const _`   |

不過兩者還是有差，輸入輸出安全性並不區分可不可變。在 Rust 的掌控之外，作業系統資源能以各種形式共享，所以輸入輸出可以視為使用了[內部可變性]。

[內部可變性]: https://doc.rust-lang.org/reference/interior-mutability.html

## `AsFd`、`Into<OwnedFd>` 與 `From<OwnedFd>`

這三個型別概念上，在大多數用例中分別取代 `AsRawFd::as_raw_fd`、`IntoRawFd::into_raw_fd`，以及 `FromRawFd::from_raw_fd`。它們依據 `OwnedFd` 及 `BorrowedFd` 來運作，所以也會自動實施其輸入輸出安全性不變的規則。

使用這些型別後，就能避免在[動機]一節的 `do_some_io` 範例中提及的問題。由於只有合理擁有或借用檔案描述子的型別能實作 `AsFd`，所以這個版本的 `do_some_io` 不需要擔心偽造或迷失（dangling）的檔案描述子傳入。

```rust
pub fn do_some_io<FD: AsFd>(input: &FD) -> io::Result<()> {
    some_syscall(input.as_fd())
}
```

至於在 Windows 上，會以 `Handle` 和 `Socket` 形式呈現相應的型別。

## 漸進式採用

輸入輸出安全性及其新型別與新特徵並不需要一次性全面導入，而是能夠分階段漸進採用：

- 首先，在 `std` 新增這些新型別與新特徵，並替相關的型別實作之。這是向下相容的改變。
- 在此之後，crate 可以開始用這些新型別，並替 crate 自己的型別實作這些新特徵。這些改變相對小，而且符合語意化版號相容性，不需其他特殊處理。
- 當標準函式庫和夠多的熱門 crate 實作這些新特徵後，其他 crate 可按照它們的開發步調，將這些新特徵作為泛型引數的限定條件（bound)。雖然這些改變不符合語意化版號的相容性，不過多數 API 使用者改用新特徵時並不需改變程式碼。

# 技術文件式解說
[技術文件式解說]: #技術文件式解說

## 輸入輸出安全性概念

Rust 語言除了有記憶體安全性之外，Rust 標準函式庫同時提供了對輸入輸出安全性的保證。一個合法的輸入輸出操作，其所操作的原始 handle（[`RawFd`]、 [`RawHandle`]、[`RawSocket`]）必為明確從作業系統回傳之值，且這些操作僅發生在與作業系統關聯之生命週期內。當一段 Rust 程式碼宣稱輸入輸出安全性，代表該程式碼不可能導致非法的輸入輸出操作。

雖然有些作業系統的文件中說明其檔案描述子的配置演算法，但從這些演算法旁敲側擊出來的 handle 值並不會視為「明確從作業系統回傳之值」。

對接受任意原始輸入輸出 handle 值（[`RawFd`]、[`RawHandle`]、[`RawSocket`]）的函式，若安全的 API 會藉由這些 handle 執行輸入輸出，應標示為 `unsafe`。

## `OwnedFd` 與 `BorrowedFd<'fd>`

`OwnedFd` 與 `BorrowedFd` 皆為 `repr(transparent)`，並帶有一個 `RawFd` 值，且兩者皆可應用區位最佳化（niche optimizations），所以和 `Option<OwnedFd>` 與 `Option<BorrowedFd<'_>>` 的大小相同，而且可在 FFI 宣告的函式中使用，例如 `open`, `read`, `write`, `close` 等。若以上述方法使用之，它們將確保 FFI 邊界的輸入輸出安全性。

這些型別同時會實作既有的 `AsRawFd`、`IntoRawFd`，以及 `FromRawFd` 特徵，所以它們可和既有程式碼的 `RawFd` 型別交互使用。

## `AsFd`、`Into<OwnedFd>` 與 `From<OwnedFd>`

這些型別提供 `as_fd`、`into` 與 `from` 函式，類似於 `AsRawFd::as_raw_fd`、`IntoRawFd::into_raw_fd` 與 `FromRawFd::from_raw_fd`。

## 原型實作

上述所有原型放在：

<https://github.com/sunfishcode/io-lifetimes>

README.md 有文件鏈結、範例、和當前提供類似功能的 crate 之調查研究。

# 缺點
[缺點]: #缺點

Crate 若用到檔案描述子，如 [`nix`] 或 [`mio`]，將需要遷移到有實作 `AsFd` 的型別，或將這類函式標示為不安全。

crates 若用 `AsRawFd` 或 `IntoRawFd` 來接收任何「類檔案」或「類 socket」型別，如 [`socket2`] 的 [`SockRef::from`]，將需換成 `AsFd` 或 `Into<OwnedFd>`，或將這類函式標示為不安全。

# 原理及替代方案
[原理及替代方案]: #原理及替代方案

## 有關「unsafe 僅為了記憶體安全性」

Rust 有個慣例：`unsafe` 只適用於標示記憶體安全性。舉個有名的案例， [`std::mem::forget`] 曾標示為不安全，但後來[被改為安全]，且其結論指出 unsafe 僅該用作標示記憶體安全性，而不該用來標示可能作繭自縛的情況或恫嚇應避免使用的 API 上。

記憶體安全性造成的危害比其他程式設計面向更甚，它不僅要避免意外行為發生，且仍需避免無法限制一段程式碼能做什麼。

輸入輸出安全性也落在這個範疇，有兩個因素：

- （若作業系統存在 `mmap` 相關 API）在安全封裝的 `mmap` 中，輸入輸出安全性的錯誤仍會導致記憶體安全性的錯誤。
- 輸入輸出安全性之錯誤也意味著一段程式碼可以在沒有通知或被授予任何引用的情形下，讀寫或刪除程式其他部分正在使用的資料。這使得在無法通曉一個 crate 鏈結到的其他 crate 的所有實作細節下，非常難以限制這個 crate 可以做什麼。

原始 handle 更像指向單獨的位址空間（address space）的原始指標，它們可能迷失（dangle）或造假。輸入輸出安全性近似於記憶體安全性，兩者皆竭力杜絕遠處來的詭異行為（spooky-action-at-a-distance），且對兩者來說，所有權都可作為建立穩健抽象化的主要根基，所以自然而然共用了相似的安全性概念。

[`std::mem::forget`]: https://doc.rust-lang.org/std/mem/fn.forget.html
[被改為安全]: https://rust-lang.github.io/rfcs/1066-safe-mem-forget.html

## 將輸入輸出 Handle 當作純資料

主要的替代方案的說法是原始 handle 為純資料（plain data），並沒有輸入輸出安全性，和作業系統資源的生命週期也無任何與生俱來的關係。至少在類 Unix 平台，這些永遠不會導致記憶體不安全或是未定義行為。

不過，大部分的 Rust 程式碼不直接與原始 handle 互動。撇開本 RFC 不談，不與原始 handle 互動是件好事。所謂資源，一定會有生命週期，若大部分的 Rust 程式碼能使用各方面都更易上手又能自動管理生命週期的更高階的型別，這樣鐵定更棒。不過，純資料的方案對於相對不常見的案例，最多只能讓原始 handle 的操作容易撰寫些。這可能只是蠅頭小利，甚至可能是個缺點，有可能最後變相鼓勵大家在不需要時去用了原始 handle 。

純資料的方案亦不需要變更任何 crate 的程式碼。而輸入輸出安全性的方案則需改動如 [`socket2`]、[`nix`] 和 [`mio`] 這些用到 [`AsRawFd`] 和 [`RawFd`] 的 crate，不過這改動可以漸進推廣到整個生態系，而不必一次性完成。

## `IoSafe` 特徵（與它的前身 `OwnsRaw`）

這個 RFC 在早先幾個版本提議過一個 `IoSafe` 特徵，這個特徵會帶來小程度但具侵入性的修復。來自該 RFC 的回饋促使一系列新型別與特徵的開發。這個開發牽扯更廣的 API 範圍，也意味著需要更多設計和審核。並且隨著時間推移，整個 crate 生態系需要更大規模的改動。然而，早期跡象指出，本 RFC 引入的新型別與特徵更易理解，使用上更順手且安全，所以長期來說有更穩健的基礎。

`IoSafe` 早期叫做 `OwnsRaw`。我們很難替這個特徵找到恰到好處的名字，這也許是個訊號，表示它並非良好設計的特徵。

# 先驅技術
[先驅技術]: #先驅技術

大部分記憶體安全的程式語言都對原始 handle 做了安全的抽象層。多數情況下，它們僅是簡單地避免暴露原始 handle，例如 [C#]、[Java] 等。若將透過原始 handle 執行輸入輸出標示為不安全，可讓 safe Rust 與這些程式語言達到相同程度的安全保證。

在 crates.io 上有好幾個 crate 封裝了擁有或借用的檔案描述子。[io-lifetimes 的 README.md 中 Prior Art 一節]詳細描述了其與其他既有 crate 的同異。從高層次角度來看，既有的 crate 都與 io-lifetimes 共享相同的基本概念。這些 crate 都圍繞著 Rust 生命週期與所有權概念打造，而這恰恰說明這些概念非常適合這個問題。

Android 有特殊的 API 會偵測不恰當的 `close`，詳見 rust-lang/rust#74860。這些 API 的動機就是一種輸入輸出安全性的應用。Android 的特殊 API 用了動態檢查，讓它們可以在跨來源語言的邊界實施這些規則。本 RFC 提議的輸入輸出安全性的型別和特徵只專注在 Rust 程式碼本身實施這些規則，所以它們可在編譯期間利用 Rust 的型別系統實施這些規則，而非延遲到執行期間。

[io-lifetimes 的 README.md 中 Prior Art 一節]: https://github.com/sunfishcode/io-lifetimes#prior-art
[C#]: https://docs.microsoft.com/en-us/dotnet/api/system.io.file?view=net-5.0
[Java]: https://docs.oracle.com/javase/7/docs/api/java/io/File.html?is-external=true

# 未解決問題
[未解決問題]: #未解決問題

## 形式化所有權

此 RFC 並沒有為原始 handle 的所有權和生命週期定義一個形式化模型（formal model）。這 RFC 對原始 handle 規範之定位尚不明朗。當 handle 只是整數型別時，與其關聯資源之生命週期意義為何？所有具有相同值的整數型別會共享該關連嗎？

Rust [參考手冊]根據 [LLVM 的指標別名規則]定義了記憶體的未定義行為；輸入輸出可能需要類似的 handle 別名規則。這對目前的實際需求而言似非必要，但未來可對此進行探索。

[參考手冊]: https://doc.rust-lang.org/reference/behavior-considered-undefined.html

# 未來展望
[未來展望]: #未來展望

以下包含一些從此 RFC 延伸的可能想法：

- Clippy 警吿常見的輸入輸出不安全狀況。
- 一個原始 handle 所有權形式化模型。可想像為延伸 Miri 使其揪出「關閉後使用」和「使用偽造的 handle」這類錯誤。
- 一個屬於 Rust，細緻且基於能力的安全模型（capabability-based security model）。藉由此模型提供的保證，在 safe Rust 語境下就不可能偽造出假的原始 handle 高階封裝。
- 還可替有實作 `AsFd`、`Into<OwnedFd>` 或 `From<OwnedFd>` 的型別添加一些方便的功能：
    - `from_into_fd` 函式：取得 `Into<OwnedFd>` 並將之轉為 `From<OwnedFd>`，讓使用者一步執行這些常見的轉換步驟。
    - `as_filelike_view::<T>()` 函式：回傳一個 `View`，其中包含內部檔案描述子構建出來的暫時實例 T，讓使用者能以 `File` 或 `TcpStream` 等方式查看原始的檔案描述子。
- 簡單使用情景的可攜性。由於 Windows 有兩種不同的 handle 型別，但 Unix 只有一種，因此在這領域中達成可攜性並非易事。然而，在部分案例中，可將 `AsFd` 和 `AsHandle` 一視同仁，而另外一些情況則可以把 `AsFd` 和 `AsSocket` 當作相同的。在這兩類情形，普通的 `FileLike` 和 `SocketLike` 抽象化能讓程式碼泛用在 Unix 和 Windows 上。

  類似的可攜性也能推廣到 `From<OwnedFd>` 及 `Into<OwnedFd>`。

# 致謝
[致謝]: #致謝

感謝 Ralf Jung ([@RalfJung]) 引導我理解這個主題至此，鼓勵我和審核這個 RFC 的草案，並耐心回答我諸多問題！

[@RalfJung]: https://github.com/RalfJung
[`File`]: https://doc.rust-lang.org/stable/std/fs/struct.File.html
[`TcpStream`]: https://doc.rust-lang.org/stable/std/net/struct.TcpStream.html
[`FromRawFd`]: https://doc.rust-lang.org/stable/std/os/unix/io/trait.FromRawFd.html
[`FromRawHandle`]: https://doc.rust-lang.org/stable/std/os/windows/io/trait.FromRawHandle.html
[`FromRawSocket`]: https://doc.rust-lang.org/stable/std/os/windows/io/trait.FromRawSocket.html
[`AsRawFd`]: https://doc.rust-lang.org/stable/std/os/unix/io/trait.AsRawFd.html
[`AsRawHandle`]: https://doc.rust-lang.org/stable/std/os/windows/io/trait.AsRawHandle.html
[`AsRawSocket`]: https://doc.rust-lang.org/stable/std/os/windows/io/trait.AsRawSocket.html
[`IntoRawFd`]: https://doc.rust-lang.org/stable/std/os/unix/io/trait.IntoRawFd.html
[`IntoRawHandle`]: https://doc.rust-lang.org/stable/std/os/windows/io/trait.IntoRawHandle.html
[`IntoRawSocket`]: https://doc.rust-lang.org/stable/std/os/windows/io/trait.IntoRawSocket.html
[`RawFd`]: https://doc.rust-lang.org/stable/std/os/unix/io/type.RawFd.html
[`RawHandle`]: https://doc.rust-lang.org/stable/std/os/windows/io/type.RawHandle.html
[`RawSocket`]: https://doc.rust-lang.org/stable/std/os/windows/io/type.RawSocket.html
[`AsRawFd::as_raw_fd`]: https://doc.rust-lang.org/stable/std/os/unix/io/trait.AsRawFd.html#tymethod.as_raw_fd
[`FromRawFd::from_raw_fd`]: https://doc.rust-lang.org/stable/std/os/unix/io/trait.FromRawFd.html#tymethod.from_raw_fd
[`from_raw_fd`]: https://doc.rust-lang.org/stable/std/os/unix/io/trait.FromRawFd.html#tymethod.from_raw_fd
[`from_raw_handle`]: https://doc.rust-lang.org/stable/std/os/windows/io/trait.FromRawHandle.html#tymethod.from_raw_handle
[`from_raw_socket`]: https://doc.rust-lang.org/stable/std/os/windows/io/trait.FromRawSocket.html#tymethod.from_raw_socket
[`std::mem::forget`]: https://doc.rust-lang.org/stable/std/mem/fn.forget.html
[`SockRef::from`]: https://docs.rs/socket2/0.4.0/socket2/struct.SockRef.html#method.from
[`unsafe_io::OwnsRaw`]: https://docs.rs/unsafe-io/0.6.2/unsafe_io/trait.OwnsRaw.html
[LLVM 的指標別名規則]: http://llvm.org/docs/LangRef.html#pointer-aliasing-rules
[`nix`]: https://crates.io/crates/nix
[`mio`]: https://crates.io/crates/mio
[`socket2`]: https://crates.io/crates/socket2
[`unsafe-io`]: https://crates.io/crates/unsafe-io
[`posish`]: https://crates.io/crates/posish
[rust-lang/rust#76969]: https://github.com/rust-lang/rust/pull/76969
[`memfd_create`]: https://man7.org/linux/man-pages/man2/memfd_create.2.html
