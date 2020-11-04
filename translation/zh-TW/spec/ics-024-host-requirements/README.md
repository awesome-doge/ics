---
ics: 24
title: 主機狀態機要求
stage: 草案
category: IBC/TAO
kind: 介面
requires: 23
required-by: 2, 3, 4, 5, 18
author: Christopher Goes <cwgoes@tendermint.com>
created: 2019-04-16
modified: 2019-08-25
---

## 概要

該規範定義了必須提供的最小介面集和滿足運行區塊鏈間鏈通信協議的狀態機必須實現的屬性。

### 動機

IBC 被設計為通用標準，將由各種區塊鏈和狀態機運行，並且必須明確定義主機的要求。

### 定義

### 所需屬性

IBC 應該要求底層狀態機提供儘可能簡單的介面，以最大程度的簡化正確的實現。

## 技術指標

### 模組系統

主機狀態機必須支持一個模組系統，這樣，獨立的，可能相互不信任的代碼包就可以安全的在同一帳本上執行，控制它們如何以及何時允許其他模組與其通信，並由“主模組”或執行環境識別這些程式碼。

IBC/TAO 規範定義了兩個模組的實現：核心“ IBC 處理程序”模組和“ IBC 中繼器”模組。 IBC/APP 規範還為特定的封包處理應用程式邏輯定義了其他模組。 IBC 要求可以使用“主模組”或執行環境來授予主機狀態機上的其他模組對 IBC 處理程序模組和/或 IBC 路由模組的訪問權限，但不對處於同個狀態機上的任何其他模組的功能或通信能力施加任何要求。

### 路徑，標識符，分隔符

`Identifier`是一個位元組字串，用作狀態存儲的對象（例如連接，通道或輕用戶端）的鍵。

標識符必須為非空（正整數長度）。

標識符只能由以下類別之一中的字元組成：

- 字母數字
- `.`, `_`, `+`, `-`, `#`
- `[`, `]`, `<`, `>`

`Path`是用作狀態存儲對象的鍵的位元組串。路徑必須僅包含標識符，常量字串和分隔符`"/"` 。

標識符並不意圖成為有價值的資源，以防止名字搶註，可能需要實現最小長度要求或偽隨機生成，但本規範未施加特別限制。

分隔符`"/"`用於分隔和連接兩個標識符或一個標識符和一個常量位元組串。標識符不得包含`"/"`字元，以防止歧義。

在整個說明書中，大括號表示的變數插值，用作定義路徑格式的簡寫，例如`client/{clientIdentifier}/consensusState` 。

除非另有說明，否則本規範中列出的所有標識符和所有字串都必須編碼為 ASCII。

### 鍵/值存儲

主機狀態機必須提供鍵/值存儲介面，具有以標準方式運行的三個函數：

```typescript
type get = (path: Path) => Value | void
```

```typescript
type set = (path: Path, value: Value) => void
```

```typescript
type delete = (path: Path) => void
```

`Path`如上所述。 `Value`是特定數據結構的任意位元組串編碼。編碼細節留給單獨的 ICS。

這些函數必須僅能由 IBC 處理程序模組（在單獨的標準中描述了其實現）使用，因此只有 IBC 處理程序模組才能`set`或`delete` `get`可以讀取的路徑。這可以實現為整個狀態機中的一個大型的鍵/值存儲的子存儲（前綴鍵空間）。

主機狀態機務必提供此介面的兩個實例- 一個`provableStore`用於供其他鏈讀取的存儲和主機的本地存儲`privateStore`，在這`get` ， `set`和`delete`可以被調用，例如`provableStore.set('some/path', 'value')` 。

對於`provableStore` ：

- 寫入鍵/值存儲的數據必須可以使用 [ICS 23](../ics-023-vector-commitments) 中定義的向量承諾從外部證明。
- 必須使用這些規範中提供的，如 proto3 文件中的典範數據結構編碼。

對於`privateStore` ：

- 可以支持外部證明，但不是必須的-IBC 處理程序將永遠不會向其寫入需要證明的數據。
- 可以使用典範的 proto3 數據結構，但不是必須的-它可以使用應用程式環境首選的任何格式。

> 注意：任何提供這些方法和屬性的鍵/值存儲介面都足以滿足 IBC 的要求。主機狀態機可以使用路徑和值映射來實現“代理存儲”，這些映射不直接匹配透過存儲介面設置和檢索的路徑和值對-路徑可以分組在存儲桶結構中，值可以分組存儲在頁結構中，這樣就可以是用一個單獨的承諾來證明，可以以某種雙射的方式非連續的重新映射路徑空間，等等—只要`get` ， `set`和`delete`行為符合預期，並且其他機器可以在可證明的存儲中驗證路徑和值對的承諾證明（或它們的不存在）。如果適用，存儲必須對外公開此映射，以便用戶端（包括中繼器）可以確定存儲的布局以及如何構造證明。使用這種代理存儲的機器的用戶端也必須了解映射，因此它將需要新的用戶端類型或參數化的用戶端。

> 注意：此介面不需要任何特定的存儲後端或後端數據布局。狀態機可以選擇使用根據其需求配置的存儲後端，只要頂層的存儲滿足指定的介面並提供承諾證明即可。

### 路徑空間

目前，IBC/TAO 為`provableStore`和`privateStore`建議以下路徑前綴。

協議的未來版本中可能會使用未來的路徑，因此可證明存儲中的整個鍵空間必須為 IBC 處理程序保留。

可證明存儲中使用的鍵可以安全的根據每個用戶端類型而變化，只要在這裡定義的鍵格式以及在機器實現中實際使用的定義之間存在雙向映射 。

只要 IBC 處理程序對所需的特定鍵具有獨占訪問權，私有存儲的某些部分就可以安全的用於其他目的。 只要在此定義的鍵格式和實際私有存儲實現中格式之間存在雙向映射，私有存儲中使用的鍵就可以安全的變化。

請注意，下面列出的與用戶端相關的路徑反映了 [ICS 7](../ics-007-tendermint-client) 中定義的 Tendermint 用戶端，對於其他用戶端類型可能有所不同。

存儲 | 路徑格式 | 值格式 | 定義在
--- | --- | --- | ---
provableStore | "clients/{identifier}/clientType" | ClientType | [ICS 2](../ics-002-client-semantics)
privateStore | "clients/{identifier}/clientState" | ClientState | [ICS 2](../ics-007-tendermint-client)
provableStore | "clients/{identifier}/consensusState/{height}" | ConsensusState | [ICS 7](../ics-007-tendermint-client)
privateStore | "clients/{identifier}/connections | []Identifier | [ICS 3](../ics-003-connection-semantics)
provableStore | "connections/{identifier}" | ConnectionEnd | [ICS 3](../ics-003-connection-semantics)
privateStore | "ports/{identifier}" | CapabilityKey | [ICS 5](../ics-005-port-allocation)
provableStore | "channelEnds/ports/{identifier}/channels/{identifier}" | ChannelEnd | [ICS 4](../ics-004-channel-and-packet-semantics)
provableStore | "seqSends/ports/{identifier}/channels/{identifier}/nextSequenceSend" | uint64 | [ICS 4](../ics-004-channel-and-packet-semantics)
provableStore | "seqRecvs/ports/{identifier}/channels/{identifier}/nextSequenceRecv" | uint64 | [ICS 4](../ics-004-channel-and-packet-semantics)
provableStore | "seqAcks/ports/{identifier}/channels/{identifier}/nextSequenceAck" | uint64 | [ICS 4](../ics-004-channel-and-packet-semantics)
provableStore | "commitments/ports/{identifier}/channels/{identifier}/packets/{sequence}" | bytes | [ICS 4](../ics-004-channel-and-packet-semantics)
provableStore | "acks/ports/{identifier}/channels/{identifier}/acknowledgements/{sequence}" | bytes | [ICS 4](../ics-004-channel-and-packet-semantics)

### 模組布局

以空間表示，模組的布局及其在主機狀態機上包含的規範如下所示（Aardvark，Betazoid 和 Cephalopod 是任意模組）：

```
+----------------------------------------------------------------------------------+
|                                                                                  |
| Host State Machine                                                               |
|                                                                                  |
| +-------------------+       +--------------------+      +----------------------+ |
| | Module Aardvark   | <-->  | IBC Routing Module |      | IBC Handler Module   | |
| +-------------------+       |                    |      |                      | |
|                             | Implements ICS 26. |      | Implements ICS 2, 3, | |
|                             |                    |      | 4, 5 internally.     | |
| +-------------------+       |                    |      |                      | |
| | Module Betazoid   | <-->  |                    | -->  | Exposes interface    | |
| +-------------------+       |                    |      | defined in ICS 25.   | |
|                             |                    |      |                      | |
| +-------------------+       |                    |      |                      | |
| | Module Cephalopod | <-->  |                    |      |                      | |
| +-------------------+       +--------------------+      +----------------------+ |
|                                                                                  |
+----------------------------------------------------------------------------------+
```

### 共識狀態檢視

主機狀態機必須提供使用`getCurrentHeight`檢視其當前高度的功能：

```
type getCurrentHeight = () => uint64
```

主機狀態機必須使用典範的二進位制序列化定義滿足 [ICS 2](../ics-002-client-semantics) 要求的一個唯一`ConsensusState`類型。

主機狀態機必須提供使用`getConsensusState`檢視其共識狀態的能力：

```typescript
type getConsensusState = (height: uint64) => ConsensusState
```

`getConsensusState`必須至少返回連續`n`個最近的高度的共識狀態，其中`n`對於主機狀態機是恆定的。大於`n`高度可能會被安全的刪除掉（之後對這些高度的調用會失敗）。

主機狀態機必須提供使用`getStoredRecentConsensusStateCount`檢視最近`n`個共識狀態的能力 ：

```typescript
type getStoredRecentConsensusStateCount = () => uint64
```

### 承諾路徑檢視

主機鏈必須提供通過`getCommitmentPrefix`檢視其承諾路徑的能力：

```typescript
type getCommitmentPrefix = () => CommitmentPrefix
```

結果`CommitmentPrefix`是主機狀態機的鍵值存儲使用的前綴。 使用主機狀態機的`CommitmentRoot root`和`CommitmentState state` ，必須保留以下屬性：

```typescript
if provableStore.get(path) === value {
  prefixedPath = applyPrefix(getCommitmentPrefix(), path)
  if value !== nil {
    proof = createMembershipProof(state, prefixedPath, value)
    assert(verifyMembership(root, proof, prefixedPath, value))
  } else {
    proof = createNonMembershipProof(state, prefixedPath)
    assert(verifyNonMembership(root, proof, prefixedPath))
  }
}
```

對於主機狀態機， `getCommitmentPrefix`的返回值必須是恆定的。

### 時間戳訪問

主機鏈必須提供當前的 Unix 時間戳，可通過`currentTimestamp()`訪問：

```typescript
type currentTimestamp = () => uint64
```

為了在超時機制中安全使用時間戳，後續區塊頭中的時間戳必須不能是遞減的。

### 埠系統

主機狀態機必須實現一個埠系統，其中 IBC 處理程序可以允許主機狀態機中的不同模組綁定到唯一命名的埠。埠使用`Identifier`標示 。

主機狀態機必須實現與 IBC 處理程序的權限交互，以便：

- 模組綁定到埠後，其他模組將無法使用該埠，直到該模組釋放它
- 單個模組可以綁定到多個埠
- 埠的分配是先到先得的，已知模組的“預留”埠可以在狀態機第一次啟動的時候綁定。

可以透過每個埠的唯一引用（對象能力）（例如 Cosmos SDK），源身份驗證（例如以太坊）或某種其他訪問控制方法（在任何情況下，由主機狀態機實施）來實現此許可。 詳細訊息，請參見 [ICS 5](../ics-005-port-allocation) 。

希望利用特定 IBC 特性的模組可以實現某些處理程式功能，例如，向和另一個狀態機上的相關模組的通道握手過程添加其他邏輯。

### 數據報提交

實現路由模組的主機狀態機可以定義一個`submitDatagram`函數，以將[數據報](../../ibc/1_IBC_TERMINOLOGY.md) （將包含在交易中）直接提交給路由模組（在 [ICS 26](../ics-026-routing-module) 中定義）：

```typescript
type submitDatagram = (datagram: Datagram) => void
```

`submitDatagram`允許中繼器進程將 IBC 數據報直接提交到主機狀態機上的路由模組。主機狀態機可能要求提交數據報的中繼器進程有一個帳戶來支付交易費用，在更大的交易結構中對數據報進行簽名，等等— `submitDatagram`必須定義並構造任何打包所需的結構。

### 異常系統

主機狀態機務必支持異常系統，藉以使交易可以中止執行並回滾以前進行的狀態更改（包括同一交易中發生的其他模組中的狀態更改），但不包括耗費的 gas 和費用，和違反系統不變數導致狀態機掛起的行為。

這個異常系統必須暴露兩個函數： `abortTransactionUnless`和`abortSystemUnless` ，其中前者回滾交易，後者使狀態機掛起。

```typescript
type abortTransactionUnless = (bool) => void
```

如果傳遞給`abortTransactionUnless`的布爾值為`true` ，則主機狀態機無需執行任何操作。如果傳遞給`abortTransactionUnless`的布爾值為`false` ，則主機狀態機務必中止交易並回滾任何之前進行的狀態更改，但不包括消耗的 gas 和費用。

```typescript
type abortSystemUnless = (bool) => void
```

如果傳遞給`abortSystemUnless`的布爾值為`true` ，則主機狀態機無需執行任何操作。如果傳遞給`abortSystemUnless`的布爾值為`false` ，則主機狀態機務必掛起。

### 數據可用性

為了達到發送或超時的安全（deliver-or-timeout safety）保證，主機狀態機務必具有最終的數據可用性，以便中繼器最終可以獲取狀態中的任何鍵/值對。而對於僅一次安全（exactly-once safety），不需要數據可用性。

對於封包中繼的活性，主機狀態機必須具有交易活性（並因此必須具有共識活性），以便在一個高度範圍內確認進入的交易（具體就是，小於分配給封包的超時高度）。

IBC 封包數據，以及未直接存儲在狀態向量中但中繼器依賴的其他數據，必須可供中繼器進程使用並高效地計算。

具有特定共識算法的輕用戶端可能具有不同和/或更嚴格的數據可用性要求。

### 事件日誌系統

主機狀態機必須提供一個事件日誌系統，藉此可以在交易執行過程中記錄任意數據，這些數據可以存儲，索引並隨後由執行狀態機的進程查詢。中繼器利用這些事件日誌讀取 IBC 封包數據和超時，這些數據和超時未直接存儲在鏈上狀態中（因為鏈上存儲被認為是昂貴的），而是提交簡潔的加密承諾（僅存儲該承諾）。

該系統期望具有至少一個函數用於發出日誌條目，和一個函數用於查詢過去的日誌，大概如下。

狀態機可以在交易執行期間調用函數`emitLogEntry`來寫入日誌條目：

```typescript
type emitLogEntry = (topic: string, data: []byte) => void
```

`queryByTopic`函數可以被外部進程（例如中繼器）調用，以檢索在給定高度執行的交易寫入的與查詢主題關聯的所有日誌條目。

```typescript
type queryByTopic = (height: uint64, topic: string) => []byte[]
```

也可以支持更複雜的查詢功能，並且可以允許更高效的中繼器進程查詢，但不是必需的。

## 向後相容性

不適用。

## 向前相容性

鍵/值存儲功能和共識狀態類型在單個主機狀態機的操作期間不太可能更改。

因為中繼器應該能夠更新其進程，所以`submitDatagram`會隨著時間而變化。

## 範例實現

即將到來。

## 其他實現

即將到來。

## 歷史

2019年4月29日-初稿

2019年5月11日-將“ RootOfTrust”重命名為“ ConsensusState”

2019年6月25日-使用“埠”代替模組名稱

2019年8月18日-修訂模組系統，定義

## 版權

本文中的所有內容均根據 [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0) 獲得許可。
