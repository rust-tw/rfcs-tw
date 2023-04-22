- Feature Name: leadership-council
- Start Date: 2022-08-01
- RFC PR: [rust-lang/rfcs#3392](https://github.com/rust-lang/rfcs/pull/3392)
- Rust Issue: N/A
- Commit: [The commit link this page based on](https://github.com/rust-lang/rfcs/blob/075f4b30f5c33315163c8e6a75e3210af6229ded/text/3392-leadership-council.md)

# 摘要

This RFC establishes a Leadership Council as the successor of the core team[^core] and the new governance structure through which Rust Project members collectively confer the authority[^authority] to ensure successful operation of the Project. The Leadership Council delegates much of this authority to teams (which includes subteams, working groups, etc.[^teams]) who autonomously make decisions concerning their purviews. However, the Council retains some decision-making authority, outlined and delimited by this RFC.

The Council will be composed of representatives delegated to the Council from each [top-level team][top-level-teams].

The Council is charged with the success of the Rust Project as a whole. The Council will identify work that needs to be done but does not yet have a clear owner, create new teams to accomplish this work, hold existing teams accountable for the work in their purview, and coordinate and adjust the organizational structure of Project teams.

# 大綱

- [參考資料](#參考資料)
- [動機](#動機)
- [理事會之職責、期待與限制](#理事會之職責、期待與限制)
- [理事會組織架構](#理事會組織架構)
  - [一級團隊單位](#一級團隊單位)
    - [一級團隊初始名單](#一級團隊初始名單)
    - [「啟動台」 一級團隊](#啟動台-一級團隊)
    - [廢除一級團隊](#廢除一級團隊)
  - [候補與放棄代表權利](#候補與放棄代表權利)
  - [任期限制](#任期限制)
  - [對單一公司或實體代表之限制](#對單一公司或實體代表之限制)
  - [候選資格](#候選資格)
  - [與 Core 團隊之關係](#與-core-團隊之關係)
  - [與 Rust 基金會之關係](#與-rust-基金會之關係)
- [理事會決策流程](#理事會決策流程)
  - [營運與政策決策](#營運與政策決策)
  - [重複與例外](#重複與例外)
  - [共識決策流程](#共識決策流程)
    - [核准標準](#核准標準)
  - [變更與調整決策流程](#變更與調整決策流程)
  - [議程與待辦事項](#議程與待辦事項)
  - [解決僵局](#解決僵局)
  - [回饋與評估](#回饋與評估)
- [決策流程之透明度和監督](#決策流程之透明度和監督)
  - [理事會可在內部做出之決策](#理事會可在內部做出之決策)
  - [理事會須私下保密做出之決策](#理事會須私下保密做出之決策)
  - [理事會須經公開提案做出之決策](#理事會須經公開提案做出之決策)
  - [利益衝突](#利益衝突)
  - [確定與更改團隊權責](#確定與更改團隊權責)
- [監督與問責機制](#監督與問責機制)
  - [確保能追究理事會之責任](#確保能追究理事會之責任)
  - [確保能追究理事會代表之責任](#確保能追究理事會代表之責任)
  - [確保能追究團隊之責任](#確保能追究團隊之責任)
- [審核、分歧與衝突](#審核、分歧與衝突)
  - [團隊間之分歧](#團隊間之分歧)
  - [涉及團隊或專案成員之衝突](#涉及團隊或專案成員之衝突)
  - [審核人代表團](#審核人代表團)
  - [審核團隊之政策與程序](#審核團隊之政策與程序)
  - [稽核](#稽核)
  - [最終問責機制](#最終問責機制)
  - [涉及專案成員之審核措施](#涉及專案成員之審核措施)
  - [涉及理事會代表之衝突](#涉及理事會代表之衝突)
  - [涉及審核團隊成員之衝突](#涉及審核團隊成員之衝突)
- [本案之批准](#本案之批准)
- [附註](#附註)

# 參考資料

為了縮短此 RFC 的篇幅，不具約束性質的參考資料將以單獨文件形式呈現：

- [完整動機](3392-leadership-council/motivation.md)
  - [Further research into the needs of Project-wide governance（Inside Rust 部落格文章）](https://blog.rust-lang.org/inside-rust/2022/05/19/governance-update.html)
- [非此 RFC 之直接目標](3392-leadership-council/non-goals.md)
- [原理和替代方案](3392-leadership-council/alternatives.md)
- [對於理事會初期工作之建議](3392-leadership-council/initial-work-of-the-council.md)

# 動機

The Rust project consists of hundreds of globally distributed people, organized into teams with various purviews. However, a great deal of work falls outside the purview of any established team, and still needs to get done.

Historically, the core team both identified and prioritized important work that fell outside of team purviews, and also attempted to do that work itself. However, putting both of those activities in the same team has not scaled and has led to burnout.

The Leadership Council established by this RFC focuses on identifying and prioritizing work outside of team purviews. The Council primarily delegates that work, rather than doing that work itself. The Council can also serve as a coordination, organization, and accountability body between teams, such as for cross-team efforts, roadmaps, and the long-term success of the Project.

This RFC also establishes mechanisms for oversight and accountability between the Council as a whole, individual Council members, the moderation team, the Project teams, and Project members.

# 理事會之職責、期待與限制

高層級描述下，理事會**只**負責以下職責：

- 識別、優先排序以及追蹤由於缺乏明確負責者而無法執行的工作（而不應對負責者刻意降低優先度、置於待辦事項等的工作）。
- 委派該工作，並潛在可建立全新的（並可能為**暫時性的**）團隊來負責此工作。
- 為沒有明確負責者的**緊急**事項下達決策。
    - 此行為只能在特例情境下執行，即該決策無法委派給既有團隊也無法交付給全新團隊時。
- 為團隊、架構或流程協調橫跨專案的變更。
- 確保一級團隊能為其權責、對其它團隊以及對專案本身負責。
- 確保各個團隊能有完成其工作所需的人力與資源。
- 為 Rust 專案整體建立起官方的定位、觀點或意願。
    - 藉此能幫助降低橫跨專案的協調需求，尤其是面對長期公開投票與共識建立流程不夠實際的情境。舉例來說，要對第三方單位溝通 Rust 專案整體「 所想要的目標」時。

除了這些職責外，對理事會也有額外的期待與限制，以幫助判斷理事會是否正常運作：

- **委派工作**：理事會不應該主動執行本案沒有明確指派的工作；理事會必須委派給既有團隊或者成員與理事會不重複的全新團隊。執行工作的團隊中可以有理事會的代表，但是做為團隊成員並不屬於理事會代表的權責。
- **為了確保專案長期的流暢運作**：理事會應該要確保非緊急專案管理工作有接受優先度排序，並且能定期完成以確保專案不會累積成整個組織級別的負擔。
- **負責任**：由於理事會擁有廣泛的權力，理事會與理事會代表必須要為自己的行為負責任。他們應該要傾聽他人的回饋，並且能對他們是否可以繼續達到該職位所需的職責與期待進行主動的反思。
- **做好代表**：理事會代表不只要能代表整個專案各形式的考量，也要能盡可能代表 Rust 社群的各方面（人口分布、技術背景等等）。
- **共享負擔**：所有理事會代表都必須共享理事會職責的負擔。
- **尊重他人的權責**：理事會必須尊重委派給各團隊的權責。理事會必須與各團隊諮商並且共同合作解決問題，並且盡量不作出違背任何團隊意願的決策。
- **以良善原則行事**：理事會代表應該以 Rust 專案**整體**的利益作出決策，即便這些決策可能會與各個團隊、雇主以及其他外部單位的利益衝突。
- **保持透明**：雖然不是所有決策（以及決策的各方面）都能被公開，理事會應該要盡可能維持其決策的公開與透明。理事會也應該要確保專案的組織架構很明確透明。
- **尊重隱私**：理事會的成員絕不能為了透明性而洩漏個人或者機密情報，包含可能會意外洩漏重要資訊的周邊資訊。
- **維持健康的工作環境**：理事會代表應該要為他們的貢獻與其性質感到滿意。他們不應認為自己在理事會中的存在只是義務所為，而必須是因為他們用有意義的方式主動參與。
- **持續演化**：理事會受到期待會隨著時間演化以應對團隊、專案以及社群的演化。

理事會代表、審核團隊成員以及其它專案成員都應該能為身邊的人以及更廣泛的社群做好示範。這些職位都有對應的責任與領導，也因此這些人的行為都帶有份量而會對社群造成重大影響，也因此必須注意行使職權。選擇要行使這些職權的人都應該要知道身邊的人會以對應的高標準看待他們。

# 理事會組織架構

理事會由一群團隊代表所組成，各代表一個[一級團隊][top-level-teams]以及其子團隊。

每支一級團隊都能用各自所選的程序選出恰好一名代表。

任何一級團隊的成員或其子團隊的成員都能作為其代表。團隊必須提供子團隊成員對潛在候選給予意見與回饋的經驗。

各個代表最多代表一個一級團隊，就算他們也是其它團隊的成員。代表任何 Rust 團隊的主要責任將由其所屬的一級團隊代表負責。[^under-multiple-teams]

所有 Rust 專案團隊都必須至少隸屬於一個一級團隊。對於目前沒有母團隊的團隊，本案會建立起[「啟動台團隊」][launching-pad]作為暫時性的隸屬對象。以確保所有團隊都在理事會有對應的代表。

## 一級團隊單位
[top-level-teams]: #一級團隊單位

理事會將會透過公開政策決策建立一級團隊。一般來說，一級團隊應該要符合以下條件：
- 有對 Rust 專案具備根本重要性的權責
- 是該權責全面向的最終決策者
- 所有權責不隸屬於其它團隊權責（不能為子團隊或相似管理架構所有）
- 有著能期待無止盡持續的開放權責
- 目前為 Rust 專案中活躍的存在

一級團隊的數量必須介於 4 到 9（包含）之間，傾向介於 5 到 8 之間。這個數字能相對平衡多元性的需求以及相對小的架構，同時又在進行有效對話與共識凝聚時具備實務性。[^number-of-representatives]

當理事會建立起全新一級團隊時，該團隊則該指派一名理事會代表。[^bootstrapping-new-teams]當建立起全新的一級團隊時，理事會必須提供為何該一級團隊不能作為子團隊或者其它管理架構存在的解釋。

### 一級團隊初始名單

一級團隊的初始名單由本案使發表時列於 [rust-lang.org 網站的一級管理區域](https://www.rust-lang.org/governance)的所有團隊組成（除去 Core 團隊與校友），再加上[「啟動台」團隊][launching-pad]：
- 編譯器
- Crates.io
- 開發工具
- 基礎架構
- 語言
- 啟動台
- 函式庫
- 審核
- 發佈

本清單並不是最佳的一級團隊組成。本案建議理事會的最開始的工作就是檢視既存的管理架構並確保所有架構都在一個或多個一級團隊中有直接或間接的代表，並且確保所有一級團隊都有符合能被認定為一級團隊的條件。這將會涉及對一級團隊組成的調整。

### 「啟動台」一級團隊
[launching-pad]: #「啟動台」一級團隊

本案會建立「啟動台（launching pad）」團隊以**暫時**接受沒有對應一級團隊可以歸類的子團隊。這樣能夠確保在更加永久性的母團隊被找到或者建立起之前，所有團隊都在理事會有代表。

「啟動台」是個雨傘團隊：該團隊沒有直接成員，只有子團隊代表。

理事會應該要致力於為所有「啟動台」子團隊找尋或者建立更合適的母團隊，並且隨之將子團隊移轉至對應的母團隊。

在某些情況下，雖有合適的母團隊但尚未準備好接受子團隊時，啟動台能作為臨時歸宿容納這些例子。

而對於團隊面臨移除或重組時，如果移除或重組時並沒有明確將其子團隊納入其它組織中的母團隊，則啟動台將會作為這些子團隊的預設歸宿。

理事會必須每六個月審視「啟動台」中的子團隊成員狀況，以確保有合適的流程在協助所有子團隊成員找到新的母團隊。而與其它一級團隊相同的是，只要理事會認為「啟動台」團隊不再需要，就可以將之撤除（並且移除掉其在理事會中的代表）。撤除「啟動台」團隊的流程與其它一級團隊同理。除此之外，理事會也可以給予「啟動台」團隊自己的權責，但是此措施不屬於本案的範圍內。

### 廢除一級團隊

任何要移除團隊的一級指定（或其它會導致無法參與理事會）的決策，都需要除卻將被移除的一級團隊代表外所有理事會代表的共識。雖然如此，必須要邀請考慮移除的團隊代表參與關於團隊移除的理事會審議，而理事會也只能在極端特例下不顧團隊反對將之移除。

理事會不得移除審核團隊。理事會不能在沒有審核團隊的同意下改變審核團隊的權責。

## 候補與放棄代表權利

A representative may end their term early if necessary, such as due to changes in their availability or circumstances. The respective top-level team must then begin selecting a new representative. The role of representative is a volunteer position. No one is obligated to fill that role, and no team is permitted to make serving as a representative a necessary obligation of membership in a team. However, a representative is obligated to fulfill the duties of the position of representative, or resign that position.

A top-level team may decide to temporarily relinquish their representation, such as if the team is temporarily understaffed and they have no willing representative. However, if the team does not designate a Council representative, they forgo their right to actively participate in decision-making at a Project-wide level. All Council procedures including decision-making should not be blocked due to this omission. The Council is still obligated to consider new information and objections from all Project members. However, the Council is not obligated to block decisions to specially consider or collate a non-represented team's feedback.

Sending a representative to the Council is considered a duty of a top-level team, and not being able to regularly do so means the team is not fulfilling its duties. However, a Council representative does not relinquish their role in cases of short absence due to temporary illness, vacation, etc.

A top-level team can designate an alternate representative to serve in the event their primary representative is unavailable. This alternate assumes the full role of Council representative until the return of the primary representative. Alternate representatives do not regularly attend meetings when the primary representative is present (to avoid doubling the number of attendees).

If a team's representative *and* any alternates fail to participate in any Council proceedings for 3 consecutive weeks, the team's representative ceases to count towards the decision-making quorum requirements of the Council until the team can provide a representative able to participate. The Council must notify the team of this before it takes effect. If a team wishes to ensure the Council does not make decisions without their input or without an ability for objections to be made on their behalf, they should ensure they have an alternate representative available.

A top-level team may change their representative before the end of their term, if necessary.  However, as maintaining continuity incurs overhead, teams should avoid changing their representatives more than necessary. Teams have the primary responsibility for briefing their representative and alternates on team-specific issues or positions they wish to handle on an ongoing basis. The Council and team share the responsibilities of maintaining continuity for ongoing issues within the Council, and of providing context to alternates and other new representatives.

For private matters, the Council should exercise discretion on informing alternates, to avoid spreading private information unnecessarily; the Council can brief alternates if they need to step in.

## 任期限制

理事會代表任期為一年。對於任何已被認可的代表團（來自特定一級團隊的代表團），每位代表有最多連任三次的非硬性限制。一位代表在理事會收到來自個別團隊明確指出他們無法產生另一位代表時，得超過任期限制（舉例來說，因為缺乏其他有意願的候選人，或是因為團隊成員對其他任何候選人提出反對意見）。

除此之外，對於單一代表在其它一級團隊服務的任期數、或是在單一一級團隊的非連續任期，並沒有任何硬性限制。團隊應該力求在經驗的存續與代表的替換之間取得平衡，以提供更多人這樣的經驗。[^representative-selection]

一半的代表將在（每年的）三月底進行任命，另一半則在九月底進行任命。這避免了同時更換所有理事會代表。對於初始的理事會以及在任何時候的一級團隊組成發生改變，理事會以及一級團隊應該共同努力使任期結束日大約平均落在在三月與九月各半。然而，每個任期應該至少持續六個月（為了避免任期過短，暫時的不平衡是可被接受的）。

如果理事會及一級團隊無法就適當的任期結束日變更達成共識，代表將在三月底或九月底兩者之間被隨機分配一個任期結束日（且在至少 6 個月後）以維持平衡。

## 對單一公司或實體代表之限制

Council representatives must not disproportionately come from any one company, legal entity, or closely related set of legal entities, to avoid impropriety or the appearance of impropriety. If the Council has 5 or fewer representatives, no more than 1 representative may have the same affiliation; if the Council has 6 or more representatives, no more than 2 representatives may have the same affiliation.

Closely related legal entities include branches/divisions/subsidiaries of the same entity, entities connected through substantial ownership interests, or similar. The Council may make a judgment call in unusual cases, taking care to avoid conflicts of interest in that decision.

A Council representative is affiliated with a company or other legal entity if they derive a substantive fraction of their income from that entity (such as from an employer, client, or major sponsor). Representatives must promptly disclose changes in their affiliations.

If this constraint does not hold, whether by a representative changing affiliation, top-level teams appointing new representatives, or the Council size changing, restore the constraint as follows:
- Representatives with the same affiliation may first attempt to resolve the issue amongst themselves, such that a representative voluntarily steps down and their team appoints someone else.
  - This must be a decision by the representative, not their affiliated entity; it is considered improper for the affiliated entity to influence this decision.
  - Representatives have equal standing in such a discussion; factors such as seniority in the Project or the Council must not be used to pressure people.
- If the representatives with that affiliation cannot agree, one such representative is removed at random. (If the constraint still does not hold, the remaining representatives may again attempt to resolve the issue amongst themselves before repeating this.) This is likely to produce suboptimal results; a voluntary solution will typically be preferable.
- While a team should immediately begin the process of selecting a successor, the team's existing representative may continue to serve up to 3 months of their remaining term.
- The existing representative should coordinate the transition with the incoming representative but it is the team's choice which one is an actual representative during the up to 3 month window. There is only ever one representative from the top-level team.

## 候選資格

以下是決定理想候選人的標準，這些標準雖類似團隊領導者或共同領導者的標準，但並不完全相同。儘管團隊領導者**可能**也能成為傑出的理事會代表，但擔任團隊領導者和理事會代表都需要大量時間投入，這會促使將這些角色分配給不同的人。這些標準並非硬性要求，但可用於決定誰最適合成為團隊代表。簡而言之，代表應具備以下條件：
- 有充沛的時間與精力，以滿足理事會的需求。
- 有興趣協助處理專案營運和治理相關事務。
- 廣泛了解專案的需求，且不限於其所在團隊或積極貢獻之領域。
- 對自己的團隊需求有敏銳的感知。
- 具備代表他人的氣質和能力，超越任何私人事務。
- 能夠代表其團隊的所有觀點，不僅僅是部分觀點，也不僅僅是團隊成員同意的觀點。

儘管有些團隊目前可能沒有足夠的候選人符合這些標準，理事會應積極培養這些技能之人才，因為這些技能不僅對理事會成員資格有幫助，也對這個專案有益。

## 與 Core 團隊之關係

The Leadership Council serves as the successor to the core team in all capacities. This RFC was developed with the participation and experience of the core team members, and the Council should continue seeking such input and institutional memory when possible, especially while ramping up.

External entities or processes may have references to "the Rust core team" in various capacities. The Council doesn't use the term "core team", but the Council will serve in that capacity for the purposes of any such external references.

## 與 Rust 基金會之關係

The Council is responsible for establishing the process for selecting Project directors. The Project directors are the mechanism by which the Rust Project's interests are reflected on the Rust Foundation board.

The Council delegates a purview to the Project directors to represent the Project's interests on the Foundation Board and to make certain decisions on Foundation-related matters. The exact boundaries of that purview are out of scope for this RFC.

# 理事會決策流程
[decision-making]: #the-council-s-decision-making-process

The Leadership Council make decisions of two different types: operational decisions and policy decisions. Certain considerations may be placed on a given decision depending on its classification. However, by default, the Council will use a consent decision-making process for all decisions regardless of classification.

## 營運與政策決策

Operational decisions are made on a daily basis by the Council to carry out their aims, including regular actions taking place outside of meetings (based on established policy). Policy decisions provide general reusable patterns or frameworks, meant to frame, guide, and support operations. In particular, policy decisions can provide partial automation for operational decisions or other aspects of operations. The council defaults to the consent decision making process for all decisions unless otherwise specified in this RFC or other policy.

This RFC does not attempt to precisely define which decisions are operations versus policy; rather, they fall somewhere along a continuum. The purpose of this distinction is not to direct or constrain the council's decision-making procedures. Instead, this distinction provides guidance to the Council, and clarifies how the Council intends to record, review, and refine its decisions over time. For the purposes of any requirements or guidance associated with the operational/policy classification, anything not labeled as either operational or policy in this or future policy defaults to policy. 

## 重複與例外
[repetition-and-exceptions]: #repetition-and-exceptions

Policy decisions often systematically address what might otherwise require repeated operational decisions. The Council should strive to recognize when repeated operational decisions indicate the need for a policy decision, or a policy change. In particular, the Council should avoid allowing repeated operational decisions to constitute de facto policy.

Exceptions to existing policy cannot be made via an operational decision unless such exceptions are explicitly allowed in said policy. Avoiding ad-hoc exceptions helps avoid ["normalization of deviance"](https://en.wikipedia.org/wiki/Normalization_of_deviance).

## 共識決策流程

The Council will initially be created with a single process for determining agreement to a proposal. It is however expected that the Council will add additional processes to its toolbox soon after creation.

Consent means that no representative's requirements (and thus those of the top-level team and subteams they represent) can be disregarded. The Council hears all relevant input and sets a good foundation for working together equitably with all voices weighted equally.

The Council uses consent decision-making where instead of being asked "do you agree?", representatives are asked "do you object?". This eliminates "pocket vetoes" where people have fully reviewed a proposal but decide against approving it without giving clear feedback as to the reason. Concerns, feedback, preferences, and other less critical forms of feedback do not prevent making a decision, but should still be considered for incorporation earlier in drafting and discussion. Objections, representing an unmet requirement or need, *must* be considered and resolved to proceed with a decision.

### 核准標準

The consent decision-making process has the following approval criteria:
- Posting the proposal in one of the Leadership Council's designated communication spaces (a meeting or a specific channel).
- Having confirmation that at least N-2 Council representatives (where N is the total number of Council representatives) have fully reviewed the final proposal and give their consent.
- Having no outstanding explicit objections from any Council representative.
- Providing a minimum 10 days for feedback.

The approval criteria provides a quorum mechanism, as well as sufficient time for representatives to have seen the proposal. Allowing for two non-signoffs is an acknowledgement of the volunteer nature of the Project, based on experience balancing the speed of decisions with the amount of confirmation needed for consent and non-objection; this assumes that those representatives have had time to object if they wished to do so. (This is modeled after the process used today for approval of RFCs.)

The decision-making process can end at any time if the representative proposing it decides to retract their proposal. Another representative can always adopt a proposal to keep it alive.

If conflicts of interest result in the Council being unable to meet the N-2 quorum for a decision, the Council cannot make that decision unless it follows the process documented in [the "Conflicts of interest" section for how a decision may proceed with conflicts documented][conflicts-of-interest]. In such a case, the Council should consider appropriate processes and policies to avoid future recurrences of a similar conflict.

## 變更與調整決策流程

Using the public policy process, the Council can establish different decision-making processes for classes of decisions.

For example, the Council will almost certainly also want a mechanism for quick decision-making on a subset of operational decisions, without having to wait for all representatives to affirmatively respond. This RFC doesn't define such a mechanism, but recommends that the Council develop one as one of its first actions.

When deciding on which decision-making process to adopt for a particular class of decision, the Council balances the need for quick decisions with the importance of confidence in full alignment. Consent decision-making processes fall on the following spectrum:

- Consensus decision making (prioritizes confidence in full alignment at the expense of quick decision making): team members must review and prefer the proposal over all others, any team members may raise a blocking objection
- Consent decision making (default for the Council, balances quick decisions and confidence in alignment): team members must review and may raise a blocking objection
- One second and no objections (prioritizes quick decision making at the expense of confidence in alignment): one team member must review and support, any team member may raise a blocking objection

Any policy that defines decision-making processes must at a minimum address where the proposal may be posted, quorum requirements, number of reviews required, and minimum time delay for feedback. A lack of objections is part of the approval criteria for all decision-making processes.

If conflicts of interest prevent more than a third of the Council from participating in a decision, the Council cannot make that decision unless it follows the process documented in [the "Conflicts of interest" section for how a decision may proceed with conflicts documented][conflicts-of-interest]. (This is true regardless of any other quorum requirements for the decision-making process in use.) In such a case, the Council should consider appropriate processes and policies to avoid future recurrences of a similar conflict.

The Council may also delegate subsets of its own decision-making purviews via a public policy decision, to teams, other governance structures, or roles created and filled by the Council, such as operational lead, meeting facilitator, or scribe/secretary.

Note that the Council may delegate the drafting of a proposal without necessarily delegating the decision to approve that proposal. This may be necessary in cases of Project-wide policy that intersects the purviews of many teams, or falls outside the purview of any team. This may also help when bootstrapping a new team incrementally.

## 議程與待辦事項

The Council's agenda and backlog are the primary interface through which the Council tracks and gives progress updates on issues raised by Project members throughout the Project.

To aid in the fairness and effectiveness of the agenda and backlog, the Council must:

- Use a tool that allows Project members to submit requests to the Council and to receive updates on those requests.
- Use a transparent and inclusive process for deciding on the priorities and goals for the upcoming period. This must involve regular check-ins and feedback from all representatives.
- Strive to maintain a balance between long-term strategic goals and short-term needs in the backlog and on the agenda.
- Be flexible and adaptable and be willing to adjust the backlog and agenda as needed in response to changing circumstances or priorities.
- Regularly review and update the backlog to ensure that it accurately reflects the current priorities and goals of the Council.
- Follow a clear and consistent process for moving items from the backlog to the agenda, such as delegating responsibility to roles (e.g. meeting facilitator and scribe), and consenting to the agenda at the start of meetings. Any agenda items rejected during the consent process must have their objections documented in the published meeting minutes of the Council.

## 解決僵局

In some situations the Council might need to make an decision urgently and not feel it can construct a proposal in that time that everyone will consent to. In such cases, if everyone agrees that a timely decision they disagree with would be a better outcome than no timely decision at all, the Council may use an alternative decision-making method to attempt to resolve the deadlock. The alternative process is informal, and the council members must still re-affirm their consent to the outcome through the existing decision making process. Council members may still raise objections at any time.
 
For example, the Council can consent to a vote, then once the vote is complete all of the council members would consent to whatever decision the vote arrived to. The Council should strive to document the perceived advantages and disadvantages for choosing a particular alternative decision-making model.

There is, by design, no mandatory mechanism for deadlock resolution. If the representatives do not all consent to making a decision even if they don't prefer the outcome of that decision, or if any representative feels it is still possible to produce a proposal that will garner the Council's consent, they may always maintain their objections.

If a representative withdraws an objection, or consents to a decision they do not fully agree with (whether as a result of an alternative decision-making process or otherwise), the Council should schedule an evaluation or consider shortening the time until an already scheduled evaluation, and should establish a means of measuring/evaluating the concerns voiced. The results of this review are intended to determine whether the Council should consider changing its prior decision.

## 回饋與評估

All policy decisions should have an evaluation date as part of the policy. Initial evaluation periods should be shorter in duration than subsequent evaluation periods. The length of evaluation periods should be adjusted based on the needs of the situation. Policies that seem to be working well and require few changes should be extended so less time is spent on unnecessary reviews. Policies that have been recently adjusted or called into question should have shortened evaluation periods to ensure they're iterating towards stability more quickly. The Council should establish standardized periods for classes of policy to use as defaults when determining periods for new policy. For instance, roles could have an evaluation date of 3 months initially then 1 year thereafter, while general policy could default to 6 months initially and 2 years thereafter.

- New policy decisions can always modify or replace existing policies.
- Policy decisions must be published in a central location, with version history.
- Modifications to the active policy docs should include or link to relevant context for the policy decision, rather than expecting people to find that context later.

# 決策流程之透明度和監督

由領導理事會作出的決策，根據其決策的性質會需要不同等級的透明度和監督。本段落將會給予指引，提供理事會對其決策尋求監督的手段，以及怎樣的決策會在私下或公開執行。

本案將會為特定的決策分類。所有沒有被特別提到的決策都必須使用公開決策流程。理事會可以透過[公開政策流程](decisions-that-the-Council-must-make-via-public-proposal)為分類方式進行演變。

由理事會作出的決策會根據其監督的可能性與必要性被分為三種分類之一：

- 理事會可在內部做出之決策
- 理事會須私下保密做出之決策
- 理事會須經公開提案做出之決策

## 理事會可在內部做出之決策

某些類型的營運決策可以在理事會內部做出，只要理事會存有機制讓社群在決策做出後能給予回饋。

要將全新的決策加入理事會可以內部做出的決策清單，必須透過公開政策決策進行。任何會影響架構、決策者或者對理事會監督的決策都不應該加入此清單。

理事會應該要致力於避免使用重複的內部決策建立起不成文的政策，藉此來避免進行公開提案。請見[重複與例外][repetition-and-exceptions]的詳細資訊。

本清單會明列所有理事會可以內部做出的決策：

- 決定展開一個會公開進行的流程（如「我們來建立並貼出問卷調查」、「我們來撰寫此未來公開決策的提案」）。
- 表達與溝通 Rust 專案的官方定位宣言。
- 對另一個實體（如 Rust 基金會）直接表達與溝通 Rust 專案的定位。
- 藉由 Rust 專案的溝通資源進行溝通（透過網誌或者所有 @ 標註）。
- 執行關於理事會內部流程的大多數營運決策，包含理事會的協調方式、使用來溝通的平台、見面的地點與時機、用於做出並記錄決策的範本（需求由本案它處規定）。
- 為了帶領／組織會議、紀錄與發佈記錄、從各個單位取得並整理回饋等等而指派執行官。[^council-roles]注意這些職位（頭銜、責任以及目前的在任者）都必須要被公開揭曉與記錄。
- 邀請理事會代表以外的特定人士參與特定的理事會會議或者討論，又或者主持開放給更廣闊社群的會議。（特別是鼓勵理事會邀請特定決策的受影響者參與該決策進行的會議或討論。）
- 為一個或多支團隊於其正常職權範圍內請求理事會不經公開提案做出的決策。（注意團隊可以選擇請求理事會的意見而無須請求理事會做出決策。）
- 為團隊職權重疊或模糊的情境做出一次性的判斷（但是**改變**團隊職權時一定要經過公開政策決策）。
- 任何此案或者未來理事會政策指定為營運決策的決策。

請見[監督與問責機制][accountability]以了解理事會決策的回饋機制。

## 理事會須私下保密做出之決策

有些決策必然會涉及關於個人或者其它實體的隱私細節，公開這些細節會對個人或實體和專案造成負面影響（如個人與實體的安全性，和對專案的信任）。

這項額外限制應該要被視為特例。並不會允許私下做出[下一段落會提到需要公開提案的決策][decisions-that-the-Council-must-make-via-public-proposal]。但是可以允許理事會於內部做出的決策能維持其隱私性，而不需要提供公開監督完整的資訊。

理事會也可以拒絕私下做出決策，如理事會認定不屬權責內事項（並且選則轉交給另一個團隊），或者相信此事項必須被公開應對的時候。然而即便在這樣的狀況下，理事會仍然不能公開揭曉所機密交付的資訊（否則理事會就沒有辦法受到信任交付這些資訊）。此明顯特例是為了避免對安全造成即刻威脅而設立的。

私下決策不能被用來建立政策。理事會應該要致力於避免使用重複的內部決策建立起不成文的政策，藉此來避免公開提案。請見[重複與例外][repetition-and-exceptions]的詳細資訊。

本清單會詳盡列出理事會可以部分或完全私下做出的決策：

- 決定與新產業／開源事業的關係時，必須於推出前維持機密性時。
- 討論團隊之間涉及人際間互動／衝突的個人資訊時。
- 代表專案與第三方進行契約磋商時（如接受提供給專案的資源）。
- 涉及專案相關的爭議政治、個人安全、或者其它可能會讓人無法安全公開發言的議題時。
- 討論一個團隊或個人是否與為何需要協助與支持，可能會涉及個人資訊時。
- 任何此案或者未來理事會政策指定為私下決策的項目。

理事會可以將將其它團隊的成員拉入可能會帶來私下或公開決策的私下討論，除非這麼做會導致揭露給理事會的隱私資訊會在不被允許的情況下更大幅度曝光。可能的情況下，理事會應該要試著將會受到決策影響的人或團隊納入討論。如此一來也能提供額外的監督。

有些事項可能不適合進行完全公開的揭曉，但仍適合分享給能被信任的討論圈（例如所有專案成員、團隊領導者或者會涉及／影響的單位）。理事會應該要致力於將資訊分享給最大合適的對象。

理事會可以在未知資訊是否有敏感性時決定要保留決策或者決策的某些面向。然而，隨時間過去合適的分享對象更加明確或擴展時，理事會應該要回顧其資訊分享決策。

理事會應該讓審核團隊知曉涉及人際衝突／紛爭的事項，因為該事項屬於審核團隊的職權內，同時也能得額外的監督。

理事會應該要評估決策的那些部分或其相關討論應該私下保密，並且考慮是否有將非敏感的部分公諸於眾，而不該只因其中一部分應該要維持私下保密而將整件事保密。所能公布的可能性包含討論本身的存在，或者是自身細節不具備敏感性的一般性議題。

私下保密的事項在未來不再具備敏感性時，可以被公開或者部分公開。然而有些事項可能**永遠**都無法被公開，代表說這些事項將不可能接受更廣泛的檢視或監督。因此，理事會在做出私下決策前必須要審慎考慮。

理事會應該要盡可能不做出私下決策。理事會應該要有合適的額外流程能鼓勵代表們共同檢視如此決策並考慮其必要性。

## 理事會須經公開提案做出之決策
[decisions-that-the-Council-must-make-via-public-proposal]: #理事會須經公開提案做出之決策

在此分類中的決策需要理事會主動**提前**在決策做下前向更廣泛的 Rust 專案參與者尋求回饋。這樣的決策會透過合適的公開決策流程提案並決策，目前必須透過 RFC 流程（然而理事會未來可以採用不同的公開提案流程）。公開決策流程必須有所有代表的共識（無論是透過主動同意或無反對的形式），必須接受理事會代表的反對阻擋，必須有合理的時間接受公開評估與討論，並且必須提供公開回饋給理事會的明確途徑。

依循既有的 RFC 流程，公開提案必須要在決策生效前有個最低的延遲時間以接受回饋。任何代表都可以要求特定決策接受回饋的時長最多延長至總計 20 天。理事會可以做出內部營運決策來延長回饋期間至超過 20 天。接受回饋的延遲期間只有在通過決策的必要條件都滿足時才能開始計算，包含沒有任何提出的反對。如果在延遲期間有反對被提出並且解決，延遲等待期間將會再次重新計算。

領導理事會被期待能隨時間演化以應對團隊、Rust 專案以及社群需求的演變。如此演變的規模可大可小，即會需要對應的監督程度。會在本質上影響理事會形式的變更必須要被納入公開決策流程。

作為上述的例外，對於一個一級團隊（審核團隊除外）的調整或移除可以在對應一級團隊委派代表缺席的情況下由一致同意做出。

理事會被允許進行私下**討論**可能會最後成為公開提案或者公開揭露的內部決策，討論的敏感性使得私下討論能允許決策參與者更自由坦蕩地發言時可以如此進行。除此之外在特定例子下，無法被揭曉的隱私資訊可能會影響到一個公開決策／提案，此情況下理事會應該要致力於維持透明性與降低誤導資訊，並且避免做出所有判斷都私下進行的完全不透明決策。

注意除非有被明確指出（透過此案或者未來的公開提案），否則所有決策都隸屬於此分類，因此此清單（與其它兩個分類不同）是刻意有模糊與廣泛性：目標是在不需要明確規定的情況下提供指引怎樣的決策會隸屬於此分類。

- 任何會修改領導理事會的決策者清單或者決策流程的決策。例如：
    - 改變這個清單（或者對此案的修改）。
    - 修改用於發表與核准理事會公開提案的流程。該提案必須使用既有的流程進行，而不能使用所提案的流程進行。
    - 增加、修改或移除會影響理事會成員資格的政策。
    - 增加、修改或移除一個或多個一級團隊。其中包含：
        - 修改一個一級團隊的權責到會使之實質上成為不同團隊的程度時。
        - 重新組織專案到頂級團隊會會隸屬到其它團隊下的程度時。
    - 增加除了一級團隊委派的代表以外的理事會代表。
    - 增加、修改或移除政策關於理事會法定人數或能制定具約束力的決策所能制定的地點時。
- 任何政策決策，除了一次性的營運決策以外（見[決策相關段落][decision-making]以了解關於政策決策與營運決策差異的細節）。其中包含任何會對其它專案中的部分（如其它團隊或個人）造成約束力，實質成為所有團隊普通權責例外的決策。在此舉例一些政策決策：
    - 修改或延長既存的政策，包含過去透過 RFC 決議的政策。
    - 會影響 Rust 專案軟體或者其它 Rust 專案成果的條約／授權政策。
    - 對行為準則進行的變更。
    - 會影響 Rust 專案成員與任何團隊資格的政策。
    - 對審核團隊如何審核理事會代表與領導理事會整體的變更。此類決策必須與審核團隊共同決策。
    - 代表 Rust 專案整體對其它專案或組織做出長期承諾的協議。（涉及團隊已經同意承諾的一次性承諾則不受此限）
    - 創造或者大幅修改條約架構時（如增設額外的基金會、改變與 Rust 基金會之間的關係、與其它法律實體合作時）。
    - 根據一個或多個團隊制定隸屬於這些團隊普通權責範圍內的政策決策時。（注意團隊可以請求理事會提供意見而不需要請求理事會做出決策）
    - 決定某類型的未來決策都將永遠都由理事會做出，而不委派給其它團隊時。
- 任何本案或未來理事會政策指定為公開政策決策的決策。

## 利益衝突
[conflicts-of-interest]: #利益衝突

理事會代表不應參與或影響其具有利益衝突的決策。

前在的利益衝突包含，但不限於：
- 個人：與他們個人有關的決策
- 財務：會對該代表造成重大的財務影響
- 聘僱或同性質決策：涉及同公司的他人或者對該公司會相對其它公司造成顯著較多的好處／壞處的決策
- 專業或其它相關性：會涉及該代表相關的組織的決策，例如產業／專業／標準／政府組織
- 家庭或朋友：決策涉及對象使得無法期待該代表能公正決策時，包含用其它形式與該人相關的利益衝突（例如家庭成員的事業）

理事會代表必須及時揭曉其利益衝突並迴避會受影響的決策。理事會代表還必須每年主動向其他代表和審核團隊揭曉可能會造成潛在衝突的來源。

請注意，即使一項提案中沒有提及特定的實體，也有可能會造成利益衝突。例如，理事會代表不能利用他們的職位調整提案中的需求，使其雇主不成比例地受益。

就算一名代表的雇主或者同性質的存在偏好一項提案的大致內容，對 Rust 社群整體都偏好的提案並不會自動導致一名代表產生利益衝突，只要提案本身不偏好特定的實體。舉例來說，一項提案改善特定 Rust 元件安全性的提案，並不會因為代表們的雇主對 Rust 的安全性有偏好而產生利益衝突；然而如果一項提議會涉及特定開發者、安全專家或特定人士能預期從此提案中獲得利益時，則可能還是會造成衝突。

即便被理事會視為輕微衝突，理事會也不得無視任何有效的利益衝突。然而理事會可以評估**是否**存在衝突。理事會代表必須要提出潛在的衝突讓理事會能夠作出此判斷。

理事會可以會向迴避的代表請求特定資訊，而迴避的代表也可以在受到請求時提供該項資訊。

在可能且實際時，理事會應該要拆開決策以降低利益衝突的影響範圍。舉例來說，理事會可以將對特定類型硬體的取用決策（而不指定特定需求或選擇提供者）與具體購置硬體與購買地點的決策分開，使得特定利益衝突只會影響到後者的決策。

同時考慮 Rust 專案與任何專案團隊的利益時並不必然致使利益衝突。特指受團隊委派的代表本來就被**期待**要參與與該團隊相關的決策。

在不太可能發生的情況下，如果一項決策提案與足夠多的代表產生了利益衝突，以致其餘的代表不能滿足過去確立的法定人數要求，而決策仍然必須做出時，則可以應對的方式包含一級團隊必須為此特定決策提供備用的代表，或者（僅適用於公開決策時）理事會可以選擇繼續進行決策，同時公開記錄所有的利益衝突。（請注意，即使在記錄了衝突的情況下，繼續進行公開決策，實際上並不會消除衝突或防止對決策的影響；只會允許公眾判斷衝突是否影響了決策。完全消除衝突永遠是最好的作法。）在這種情況下，理事會應該要考慮合適的流程與政策，以避免未來再次發生類似的衝突。

## 確定與更改團隊權責

理事會可以在既存或新創的一級團隊（除審核團隊外）的權責之間移動領域或活動。即便特定一級團隊的權責可能會被該團隊進一步細分，理事會只會移動或調整一級團隊的權責。如果一個被細分的權責進行了移動，理事會則會與相關團隊合作，協調合適的下一步。此機制的使用時機只限於理事會認為既有團隊的權責過於廣泛，使得無法期待該團隊於現有架構下能夠完全實現其職責下使用。然而，當一個團隊只是**目前**缺乏資源來履行其部分職責時，則不應該如此進行。

理事會還必須負責核准一級團隊權責的擴張，並且必須在一級團隊職權的減少收到通知。此情況通常會在一個團隊自行決定他們希望擴大或減少其權責時發生。這也可能發生在部分一級團隊同意調整其之間的權責。理事會對於權責變化的認知是必要的，以確保理事會能重新指派或刻意不加以指派該項權責。

然而單獨或聯合的團隊可以在無須理事會核准之下進一步將其權責委託給子團隊。然而即便進行了委派，一級團隊仍應該對分配到的權責負起全責（換句話說，團隊有責任確保其委派能成功達成）。

理事會應該要偏好在團隊間權責移轉前與各個團隊合作找出替代方案，因為此項措施是非常重大的步驟。同時值得注意的是，這個機制的一個使用情境是將過去委託於一個不再存在功能性的團隊的權責加以轉移（例如該團隊的成員都沒有足夠時間），潛在可以於有對應時間與能力的人手到位重新建立該團隊時可以作為暫時處置。本案的此段落是刻意保留不限制理事會如何（與否）進行此諮商。

# 監督與問責機制
[accountability]: #監督與問責機制

以下是諸多理事會可以確保自身與他者能負起責任的機制。

## 確保能追究理事會之責任

理事會必須公開確保持續達到專案整體與社群對理事會的期待。其手段包含調整理事會的政策、措施與產出結果，同時在專案與社群期待不夠現實時予以充足的教育。

要達成這件事情，除了輪替代表並採納「預設的公開性」觀點以外，理事會也應該定期（會至少每季）對他們的活動進行廣大的公開溝通，同時也要評估根據理事會的職責、期待與限制為標準去評估理事會的運作狀況。

每一年，理事會都應該要向所有願意的專案成員徵求回饋以了解理事會是否有效地完成其目標，並且在一場允許並鼓勵所有專案成員主動參加的論壇中公開討論回饋。為了達到這件事，理事會與其他專案成員要討論本案與未來改版中提到的高層級職責、期待與限制，以判斷理事會是否達成其職責與義務。

除此之外，每位代表**個人**都有責任注意、指出並拒絕無法依循本案、其它理事會政策或其它方面理事會責任的情境。代表應該要致力於主動迴避「[責任分散效應](https://en.wikipedia.org/wiki/Diffusion_of_responsibility)」，即群體中會因為個別成員（有意無意地）認為其他人會負責而一同無法達成特定事項的現象。理事會也可以指派一個特定職位來負責應對並監督程序性事務，特別是負責提出程序上有[秩序問題](https://en.wikipedia.org/wiki/Point_of_order)，然而其他代表依然能做這件事情。

如果任何前述的流程中得出了理事會**並沒有**達成其義務，則理事會必須要盡快提出計畫，協助理事會改善，以利其達成目標。其方法可能是提出修改 RFC 或相似的措施、代表的輪替或者其它重大變更。任何計畫都應該對理事會或 Rust 管理整體上，如何根據去年經驗演變有明確的措施。

## 確保能追究理事會代表之責任

理事會代表應該要參與他者與其對應一級團隊（其性質則在本案範圍外）的定期回饋，以反映他們作為代表實現其職責的狀況。此回饋活動的目的是要幫助代表了解自己能如何更有效幫助專案。此回饋必須要分享給所有代表、所有代表的一級團隊與審核團隊。此項回饋應該要同時詢問代表做得好與能改善的部分。

除此之外，代表也應該要隨時開放其團隊與其他代表的私下回饋，並且應該要定期省思自己在理事會中的職位與成效。

此種回饋流程的產物絕不能被公開，以確保流程的安全與公正。理事會也應該要在結果無法帶來正向改變時反映並改變回饋流程。

如果其他理事會的成員認為一名理事會代表與其餘理事會成員並沒有辦法良好合作，則他們該與對應代表進行對話，而有必要時也該向該代表的團隊對話。理事會代表也必須要納入可以促進這些對話進行的審核／調解資源。審核可以幫助解決問題，且／或能判斷是否需要為此問題行動並且是否需要擴張應對的層級。

雖然個別團隊要如何確保能追究其代表責任並不在本案的範圍內，我們鼓勵各團隊以上述機制為靈感去規劃各自的政策與措施。

## 確保能追究團隊之責任

各團隊要定期彼此協調與合作，並且溝通自己的需求；在正常狀況下，理事會必須尊重個別團隊的自主性。

然而，理事會也作為各團隊可以共同以團隊或專案整體來追究彼此責任的手段。理事會可以：

- 請求一個團隊重新考慮一個並未納入其他團隊或者專案整體考慮的決策。
- 鼓勵各團隊建立可以定期將其他團隊納入考慮的流程。
- 確保對所有團隊權責有共同的瞭解。
- 確保各團隊都願意並且有能力實現這些權責。
- 建立新團隊將一個團隊的權責分為更能被應對的大小。

追究的流程不得具備懲罰性，同時此流程必須在受到影響團隊的主動合作下進行。

在極端情境下，如果團隊刻意選擇對專案整體不以良善原則行動時，則理事會有權力去改變團隊的權責、將部分權責轉移至其他團隊或直接移除整個團隊。此作為可以透過理事會常規的決策流程進行。（此項措施對審核團隊並不適用；請見下一段落關於理事會與審核團隊之間的問責機制）

# 審核、分歧與衝突

本段落描述了領導理事會和審核團隊在協助解決分歧和衝突方面的角色，以及這些團隊之間的互動。

分歧和衝突同樣座落在人際互動光譜上。分歧是更多關於事實和、或技術上的不一致，而衝突則是更多關於社交或關係上的合作障礙。許多互動可能同時表現出分歧和衝突的樣貌。理事會可以在分歧方面提供幫助，而衝突方面則屬於審核團隊的權責。

本案並未詳細規定一般審查政策，僅規定了與理事會互動以及理事會與審核團隊之間平衡的部分。一般審查政策超出了本案的範疇。

Rust 專案的許多工作都涉及與其他人的合作，所有人都非常關心自己的工作。人們之間的分歧是正常的，對分歧產生強烈情感也是正常的。分歧也可以成為揭示和解決問題的有力工具，理想情況下，有分歧的人可以在合作中（盡量）友好地探討這些分歧，而不升溫為人際衝突。

分歧和衝突出現的情況可能很複雜。分歧可能升溫為衝突，衝突也可能降級為分歧。如果情況中無法明確區分分歧和衝突，或者參與者對此有分歧，則應假定情況是衝突。

在發生衝突的情況下，涉及者應盡快聯繫審核團隊協助解決衝突。時間是在衝突惡化或造成更多傷害之前解決衝突的關鍵資源。

## 團隊間之分歧

在可能的情況下，團隊應嘗試自行解決分歧，如有需要可向理事會尋求協助。理事會可以做出判斷來解決分歧，但團隊需要保持良好的工作關係，以避免持續的分歧或升級為衝突。

團隊之間分歧的潛在解決途徑可能包括選擇先前討論過的選項，制定新選項，確定決策屬於哪個團隊的職責範疇，或者決定該決策在兩個團隊的權責範疇之外，交由理事會為這項工作找到一個新的歸屬。

## 涉及團隊或專案成員之衝突

涉及團隊或專案成員的衝突應盡快提交給審核團隊。理事會可以協助減少這些衝突對未決、緊急決策的影響，但審核團隊負責協助解決團隊之間或其他方面的衝突和人際問題。

個人或團隊也可以自願參與其他過程來解決衝突或人際問題，例如非約束性的外部審核。在這樣做時，個人或團隊應讓審核團隊知道情況，並尋求審核團隊對於合適的資源或方法的指導。個人或團隊不得使用會產生利益衝突的資源。

## 審核人代表團

審核團隊必須始終維護一個公開記錄的「緊急審核者」名單，這些緊急審核者必須通過審核團隊和理事會的內部共識決策得到批准。審核團隊和緊急審核團隊應分別由至少三名成員組成。緊急審核者必須：
- 既不是現任審核團隊的成員，也不是領導理事會的成員。
- 由理事會和審核團隊共同確定，受到 Rust 專案成員的廣泛信任；這通常意味著他們已經以某種方式參與專案。
- 由理事會和審核團隊共同確定，具有從事審核工作和[稽核][audits]的資格。更詳細的標準和指導方針將由審核政策確定，該政策超出了本案的範疇。
- 願意擔任緊急審核者：願意進行審核，並在審核團隊解散或無法進行工作的情況下，願意進行臨時審核工作，直到他們能夠任命新的正式審核者。（緊急審核者無需願意長期從事審核工作。）
- 願意熟悉審核政策和程序，達到審核團隊成員所期望的標準（包括任何相關培訓）。在可能的情況下，緊急審核者應獲得與審核團隊相同的培訓機會。

需要緊急審核者行動的情境會是高壓緊張的情況，而專案和理事會必須做好信任他們投身於這種情況的準備。選擇其他專案成員所熟悉和信任的人有助於降低這種情況下的緊張。

審核是一項容易耗竭的活動，個別審核者或審核團隊可能會希望遠離這項工作。注意，一個或多個個別審核者可以選擇辭職，此時審核團隊應該確定並引入新的審核者來填補任何空缺或不足；如果審核團隊要求緊急審核者成為正式審核者，該小組應該任命一位新的緊急審核者。辭職的個別審核者**可以**被選為緊急審核者。如果審核團隊整體同時無法進行工作（由理事會和緊急審核者通過內部共識決策共同確定），或選擇同時辭職，緊急審核者將成為臨時審核團隊，並必須迅速任命新的緊急審核者，並開始尋找新的正式審核者。

由於緊急審核者角色除了特殊情況以外不需要執行任何定期活動，因此被任命為該角色的人必須與審核團隊定期檢查，以確認他們仍願意擔任該角色，並避免緊急審核者突然被需要並被發現無法進行工作的情況。

## 審核團隊之政策與程序

審核團隊有責任制定完善的政策和程序。理事會提供監督和協助，以確保審核團隊擁有這些政策和程序，並且它們足夠完善。

理事會可以向審核團隊提供回饋，審核團隊有義務考慮收到的所有回饋。如果理事會認為審核團隊未遵循審核政策和程序，理事會可以[要求緊急審核者進行稽核][audits]。然而，理事會不能否決審核決定或政策。

## 稽核
[audits]: #稽核

如果任何理事會成員認為審核決定（或一系列決定）未遵循審核團隊既定的政策和程序，他們應立即通知審核團隊。理事會和審核團隊應該相互合作，討論和理解這些問題，並努力解決它們。

本案提供的一種以保密形式檢查審核團隊行為的機制，是稽核（audit）機制。在任何情況下，如果任何理事會成員認為審核團隊的行動未遵循記錄紀錄有案的政策或程序時，則該理事會成員可以決定啟動稽核流程。（特別是，理事會成員可以以此形式回應涉及審核情形的社群成員的回報。）此項機制的進行是除了以上的互動和對話之外的**額外**行動；此機制不是理事會與審核團隊之間直接溝通的替代品。

在稽核中，緊急審核團隊與審核團隊合作，確定審核團隊是否遵循既定的政策和程序。這一機制必然涉及緊急審核團隊依據自己的判斷來評估審核政策、具體證據或溝通以及相應的審核行動或建議行動。然而，稽核機制並不旨在對行動本身進行懷疑；稽核機制的重點是確定審核團隊是否按照其既定的政策和程序行事。

緊急審核者還會與理事會聯繫，了解他們可能需要的任何額外背景資訊。

審核流程和稽核都需要時間，並且必須謹慎執行。然而，理事會、緊急審核者和審核團隊都應該努力在合理的時間內相互表達他們的顧慮和期望，並保持開放的溝通管道。

緊急審核者在具有利益衝突的情況下不得參與決策或稽核。在緊急審核者被公開列為緊急審核團隊成員之前，他們不得查看提交給審核團隊的私人資訊；讓與審核團隊交流的人有機會評估潛在的顧慮或利益衝突。

與理事會和緊急審核團隊的討論可能發現，審核團隊在某個特定案例中不得不在政策上作出例外，因為政策中存在意想不到的條件，或者存在有無法納入政策的背景資訊。這是一個需要緊急審核團隊對例外的理由和決定例外必要性的流程進行額外審查的預期情境，但本質上並不是審核團隊職責的違反。

隨著稽核流程和理事會、審核團隊討論的進行，審核團隊可能會決定修改審核政策和、或更改特定審核決定或擬議決定的結果。這完全取決於審核團隊的決定。

緊急審核團隊必須將稽核結果報告給審核團隊和理事會以供他們審查。這不得包括任何可能直接或間接透露私人資訊的細節。此報告與和審核團隊的討論相輔相成，最終旨在解決理事會的顧慮。

## 最終問責機制

Rust 專案的領導理事會和審核團隊在專案內各自擁有相當大的權力。本案提供了許多工具來解決衝突。本段落概述了這些團隊相互追究責任的最終問責機制。本段落寫成之時期許將永遠不會被用到，希望所有團隊會盡一切可能在達到此情境之前解決衝突。

如果理事會認為審核團隊存在系統性問題（無論是基於緊急審核團隊的稽核報告還是其他原因），並且理事會和審核團隊無法自願達成協議來解決問題，那麼作為最後手段，理事會（經一致決定）可以同時解散自己和審核團隊。頂級團隊必須為理事會指派新的代表，緊急審核團隊成為新的臨時審核團隊。

相反地，如果審核團隊認為理事會存在系統性問題，並且理事會和審核團隊無法自願達成協議來解決問題，那麼作為最後手段，審核團隊（經一致決定）可以同時解散自己和理事會。只有在審核團隊成員至少有三人的情況下才能實施這一過程。最高層次的團隊必須為理事會指派新的代表，緊急審核團隊成為新的臨時審核團隊。

審核團隊在理事會的代表在決定解散理事會和審核團隊時必須回避，以避免利益衝突，但該代表仍需辭去職務。

被解除職務的代表和審核者在至少一年內不得在理事會或審核團隊中任職。

預設情況下，新理事會和臨時審核團隊將負責清晰地溝通過渡期。

這個機制是絕對的最後手段。最起碼此手段幾乎肯定不會產生最佳的結果。如果情勢已經升溫到造成此結果，必須要有很多事情都已經陷入**非常糟糕**的情境了，而那些負責清理善後工作的人應該努力防止這種情況再次發生。（無論是審核團隊還是理事會）表示情勢可能升溫到這個程度時，都應該被認為是一個強烈的信號，讓各方一起討論找到一個「總之不要用上那招」的方法來避免這種情境。

## 涉及專案成員之審核措施
[moderation-actions-involving-Project-members]: #涉及專案成員之審核措施

在進行審核工作過程中，審核團隊不僅需要對 Rust 社群成員採取行動，還需要對 Rust 專案成員採取行動。這些行動可能涵蓋從對話到從專案中移除之間的所有可能升溫措施階級。這使得審核團隊處於需要具備權力和信任的地位。此案旨在為審核團隊及理事會提供適當的問責和相互檢查手段。

如果審核團隊計劃對任何 Rust 專案成員實施外部可見的制裁（任何可能引起顯著缺席的行動，例如從職務中移除或從某專案空間中排除參與超過一週），則任何一方都可以通過與理事會或緊急審核者聯繫來要求進行[稽核][audits]，並且該稽核將自動獲准。

在此案批准後的第一年內，即使沒有請求也會自動執行稽核，以確保該流程正常運行。在那之後，理事會和審核團隊將共同審查並決定是否更新此條款。

當審核團隊向專案成員發出警告，或者向專案成員發出有關審核行動的通知時，該消息將提及要求稽核的選項。

應盡快將有關專案成員的衝突通知給審核團隊知曉。

## 涉及理事會代表之衝突

涉及理事會代表或替補代表的衝突遵循與涉及專案成員的衝突相同的流程。審核團隊具有與專案其他成員相同的審核代表或替補代表的能力，包括對任何外部可見制裁所需的[稽核][audits]。此項行為仍然受到與審核團隊其他決策相同的問責機制的約束。

除了既有的審核行動範圍之外，審核團隊還可以在接近最後手段的情況下，作為使用升溫措施階級中較輕微的手段，從專案中移除代表或替補代表。這些行動並不是特別針對理事會的行動，並且適用於其他 Rust 團隊。

- 審核團隊可以決定將一名代表從理事會中移除。該代表所代表的頂級團隊應立即委派一位新代表來完成剩餘的任期。
- 審核團隊可以決定阻止一名專案成員成為理事會代表。
- 審核團隊和理事會（排除受影響方）可以共同決定（作為私下保密共識決策）限制代表在理事會中的參與。 （在這種情況下，即便有代表存在利益衝突，他們也不會被排除在外，因為整個理事會都必須合作使制裁生效。如果利益衝突因此阻止了執行這些部分制裁，審核團隊始終可以選擇全面制裁，例如將之移除。）

所有這些行動都將觸發需要進行的稽核。理事會還必須被告知涉及代表或替補代表這樣的審核行動，或直接阻止人們成為代表的行動。

## 涉及審核團隊成員之衝突

涉及審核團隊成員的衝突將由餘下的審核團隊成員（扣除任何有利益衝突的成員）和緊急審核團隊**共同**處理，以提供額外的監督。審核團隊或緊急審核團隊的任何成員如果認為審核團隊內存在更多系統性問題，應與理事會磋商。緊急審核者必須稽核此決策並向理事會和審核團隊提供稽核報告。

# 本案之批准

自 2021 年 11 月以來，下列成員實質上已在專案擔任中領導角色：所有 Core 團隊成員、所有審核團隊成員、Rust Foundation 理事會（board）上的所有專案代表以及「一級」團隊的負責人：
- 編譯器
- Crates.io
- 開發工具
- 基礎架構
- 語言
- 函式庫
- 審核（已包含在上方）
- 發佈

此 RFC 將以標準 RFC 程序審批，由前述實質上的領導層成員來批准。這些成員還應代表專案中其他成員提出異議，更具體來說，團隊負責人應徵求他的團隊和子團隊的回饋。

# 附註

[^core]: Unlike in some other Open Source projects, the Rust Project's "core team" does not refer to a group that decides the technical direction of the Project. As explained in more detail elsewhere in the RFC, the Rust Project distributes decision-making to many different teams who have responsibility for their specific purview. For example, the compiler team is in charge of the Rust compiler, the language team is in charge of language evolution, etc. This is part of why this RFC discontinues use of the term "core team".

[^authority]: The term 'authority' here refers to the powers and responsibilities the Council has to ensure the success of the Rust Project. This RFC lays out the limits of these powers, so that the Council will delegate the authority it has to teams responsible for the concerns of the Project. These concerns may include - but are not limited to - product vision, day-to-day procedures, engineering decisions, mentoring, and marketing.

[^teams]: Throughout this document, "teams" includes subteams, working groups, project groups, initiatives, and all other forms of official collaboration structures within the Project. "Subteams" includes all forms of collaboration structures that report up through a team.

[^under-multiple-teams]: Subteams or individuals that fall under multiple top-level teams should not get disproportionate representation by having multiple representatives speaking for them on the Council. Whenever a "diamond" structure like this exists anywhere in the organization, the teams involved in that structure should strive to avoid ambiguity or diffusion of responsibility, and ensure people and teams know what paths they should use to raise issues and provide feedback.

[^bootstrapping-new-teams]: The Council consists only of the representatives provided to it by top-level teams, and cannot appoint new ad hoc members to itself. However, if the Council identifies a gap in the project, it can create a new top-level team. In particular, the Council can bootstrap the creation of a team to address a problem for which the Project doesn't currently have coordinated/organized expertise and for which the Council doesn't know the right solution structure to charter a team solving it. In that case, the Council could bring together a team whose purview is to explore the solution-space for that problem, determine the right solution, and to return to the Council with a proposal and charter. That team would then provide a representative to the Council, who can work with the Council on aspects of that problem and solution.

[^number-of-representatives]: This also effectively constrains the number of Council representatives to the same range. Note that this constraint is independently important.

[^representative-selection]: 最終整體來說，作為一位理事會代表是為了服務個別的團隊以及 Rust 計畫。儘管此 RFC 的作者希望這個職位能夠滿足並且吸引任何足以勝任這份工作的人，我們仍然希望它不被視為是用來爭奪地位的職位。

[^council-roles]: The Council is not required to assign such roles exclusively to Council representatives; the Council may appoint any willing Project member. Such roles do not constitute membership in the Council for purposes such as decision-making.
