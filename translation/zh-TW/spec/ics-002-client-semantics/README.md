---
ics: 2
title: 用戶端語義
stage: 草案
category: IBC/TAO
kind: 介面
requires: 23, 24
required-by: 3
author: Juwoon Yun <joon@tendermint.com>, Christopher Goes <cwgoes@tendermint.com>
created: 2019-02-25
modified: 2020-01-13
---

## 概要

該標準規定了實現區塊鏈間通信協議的狀態機共識算法所必須滿足的屬性。 這些屬性對更高層協議抽象中的有效安全的驗證而言是必需的。IBC 中用於驗證另一台狀態機的共識記錄及狀態子組件的算法稱為“合法性判定式”，並將其與驗證者認為正確的狀態配對形成“輕用戶端”（通常簡稱為“用戶端”）。

該標準還規定了如何在典範的 IBC 處理程序中存儲、註冊和更新輕用戶端。 所存儲的用戶端實例能被第三方參與者進行檢視，例如，用戶檢查鏈的狀態並確定是否發送 IBC 封包。

### 動機

在 IBC 協議中，參與者（可能是終端用戶、鏈下進程或狀態機）需要能夠對另一台狀態機共識算法認同的狀態更新進行驗證，並拒絕另一台狀態機共識算法不認同的任何潛在狀態更新。輕用戶端就是一個帶有能做到上面功能的狀態機的算法。該標準規範了輕用戶端的模型和要求，因此，只要提供滿足所列要求的相關輕用戶端算法，IBC 協議就可以輕鬆的與運行新共識算法的新狀態機集成。

除了本規範中描述的屬性外，IBC 對狀態機的內部操作及其共識算法沒有任何要求。一台狀態機可能由一個單獨的私鑰簽名進程、多個統一仲裁簽名的進程、多個運行拜占庭容錯共識算法的進程或其他尚未發明的配置組成——從 IBC 的角度來看，一個狀態機是完全由其輕用戶端的驗證和不良行為檢測邏輯來定義的。 用戶端通常不包括對狀態轉換邏輯的完整驗證（因為這將等同於簡單地又執行了一另一個狀態機），但是在特定情況下，用戶端可以選擇驗證部分狀態轉換。

用戶端還可以當作其他用戶端的閾值視角。 如果模組利用 IBC 協議與機率最終性（probabilistic-finality）共識算法進行交互，對於不同的應用可能需要不同的最終性閾值，那麼可以創建一個只寫用戶端來跟蹤不同區塊頭，多個具有不同最終性閾值（被認為是最終的狀態根後的確認深度）的只讀用戶端可以使用相同的狀態。

用戶端協議還應該支持第三方引薦。 Alice 是一台狀態機上的一個模組，希望將 Bob（Alice 認識的第二台狀態機上的第二個模組）介紹給 Carol（Alice 認識但 Bob 不認識的第三台狀態機上的第三個模組）。Alice 必須利用現有的通道傳送給 Bob 用於和 Carol 通信的典範序列化的合法性判定式，然後 Bob 可以與 Carol 建立連接和通道並直接通信。 如有必要，在 Bob 進行連接嘗試之前，Alice 還可以向 Carol 傳送 Bob 的合法性判定式，使得 Carol 獲悉並接受進來的請求。

用戶端介面也應該被構造，以便可以安全的提供自訂驗證邏輯，並在運行時定義自訂用戶端，只要基礎狀態機可以提供適當的 gas 計量機制來為計算和存儲收費。例如，在支持 WASM 執行的主機狀態機上，可以在創建用戶端實例時將合法性判定式和不良行為判定式作為可執行的 WASM 函數提供。

### 定義

- `get`, `set`, `Path`, 和 `Identifier` 在 [ICS 24](../ics-024-host-requirements) 中被定義.

- `CommitmentRoot` 如同在 [ICS 23](../ics-023-vector-commitments) 中被定義的那樣，它必須為下游邏輯提供一種廉價的方式去驗證鍵值對是否包含在特定高度的狀態中。

- `ConsensusState` 是表示合法性判定式狀態的不透明類型。`ConsensusState` 必須能夠驗證相關共識算法所達成一致的狀態更新。 它也必須以典範的方式實現可序列化，以便第三方（例如對方狀態機）可以檢查特定狀態機是否存儲了特定的共識狀態。 它最終必須能被使用它的狀態機檢視，比如狀態機可以查看某個過去高度的共識狀態。

- `ClientState` 是表示一個用戶端狀態的不透明類型。 `ClientState` 必須公開查詢函數，以驗證處於特定高度的狀態下包含或不包含鍵值對，並且能夠獲取當前的共識狀態.

### 所需屬性

輕用戶端必須提供安全的算法使用現有的`ConsensusState`來驗證其他鏈的典範區塊頭 。然後，更高級別的抽象將能夠驗證存儲在`ConsensusState`的`CommitmentRoot`的狀態的子組件確定是由其他鏈的共識算法提交的。

合法性判定式應反映正在運行相應的共識算法的全節點的行為。給定`ConsensusState`和消息列表，如果一個全節點接受由`Commit`生成的新`Header` ，那麼輕用戶端也必須接受它，如果一個全節點拒絕它，那麼輕用戶端也必須拒絕它。

由於輕用戶端不是重新執行整個消息記錄，因此在出現共識不良行為的情況下有可能輕用戶端的行為和全節點不同。在這種情況下，一個用來證明合法性判定式和全節點之間的差異的不良行為證明可以被生成，並提交給鏈，以便鏈可以安全的停用輕用戶端，使過去的狀態根無效，並等待更高級別的干預。

## 技術規範

該規範概述了每種*用戶端類型*必須定義的內容。用戶端類型是一組操作輕用戶端所需的數據結構，初始化邏輯，合法性判定式和不良行為判定式的定義。實現 IBC 協議的狀態機可以支持任意數量的用戶端類型，並且每種用戶端類型都可以使用不同的初始共識狀態進行實例化，以便進行跟蹤不同的共識實例。為了在兩台機器之間建立連接（請參閱 [ICS 3](../ics-003-connection-semantics) ）， 這些機器必須各自支持與另一台機器的共識算法相對應的用戶端類型。

特定的用戶端類型應在本規範之後的版本中定義，並且該倉庫中應存在一個典範的用戶端類型列表。 實現了 IBC 協議的機器應遵守這些用戶端類型，但他們可以選擇僅支持一個子集。

### 數據結構

#### 共識狀態

`ConsensusState` 是一個用戶端類型定義的不透明數據結構，合法性判定式用其驗證新的提交和狀態根。該結構可能包含共識過程產生的最後一次提交，包括簽名和驗證人集合元數據。

`ConsensusState` 必須由一個 `Consensus`實例生成，該實例為每個 `ConsensusState`分配唯一的高度（這樣，每個高度恰好具有一個關聯的共識狀態）。如果沒有一樣的加密承諾根，則同一鏈上的兩個`ConsensusState`不應具有相同的高度。此類事件稱為“矛盾行為”，必須歸類為不良行為。 如果發生這種情況，則應生成並提交證明，以便可以凍結用戶端，並根據需要使先前的狀態根無效。

鏈的 `ConsensusState` 必須可以被典範的序列化，以便其他鏈可以檢查存儲的共識狀態是否與另一個共識狀態相等（請參見 [ICS 24](../ics-024-host-requirements) 的鍵表）。

```typescript
type ConsensusState = bytes
```

`ConsensusState` 必須存儲在下面定義的指定的鍵下，這樣其他鏈可以驗證一個特定的共識狀態是否已存儲。

`ConsensusState` 必須定義一個 `getTimestamp()` 方法，該方法返回與該共識狀態關聯的時間戳：

```typescript
type getTimestamp = ConsensusState => uint64
```

#### 區塊頭

`Header` 是由用戶端類型定義的不透明數據結構，它提供用來更新`ConsensusState`的訊息。可以將區塊頭提交給關聯的用戶端以更新存儲的`ConsensusState` 。區塊頭可能包含一個高度、一個證明、一個加密承諾根，還有可能的合法性判定式更新。

```typescript
type Header = bytes
```

#### 共識

`Consensus` 是一個 `Header` 生成函數，它接受之前的 `ConsensusState` 和消息並返回結果。

```typescript
type Consensus = (ConsensusState, [Message]) => Header
```

### 區塊鏈

區塊鏈是一個生成有效`Header`的共識算法。它從創世`ConsensusState`開始通過各種消息生成一個唯一的區塊頭列表。

`區塊鏈` 被定義為

```typescript
interface Blockchain {
  genesis: ConsensusState
  consensus: Consensus
}
```

其中

- `Genesis`是一個創世`ConsensusState`
- `Consensus`是一個區塊頭生成函數

從`Blockchain`生成的區塊頭應滿足以下條件：

1. 每個`Header`不能有超過一個直接的孩子

- 滿足，假如：最終性和安全性
- 可能的違規場景：驗證人雙重簽名，鏈重組（在中本聰共識）

1. 每個`Header`最終必須至少有一個直接的孩子

- 滿足，假如：活性，輕用戶端驗證程序連續性
- 可能的違規場景：同步停止，不相容的硬分叉

1. 每個`Header`必須由`Consensus`生成，以確保有效的狀態轉換

- 滿足，假如：正確的塊生成和狀態機
- 可能的違規場景：不變數被破壞，超過多數驗證人共謀

除非區塊鏈滿足以上所有條件，否則 IBC 協議可能無法按預期工作：鏈可能會收到多個衝突封包，鏈可能無法從超時事件中恢復，鏈可能會竊取用戶的資產等。

合法性判定式的合法性取決於`Consensus` 的安全模型。例如， `Consensus`可以是受一個被信任的運營者管理的 PoA（proof of authority）共識，或質押價值不足的 PoS（proof of stake）共識。在這種情況下，安全假設可能被破壞， `Consensus`與合法性判定式的關聯就不存在了，並且合法性判定式的行為變的不可定義。此外， `Blockchain`可能不再滿足上述要求，這將導致區塊鏈與 IBC 協議不再相容。在這些導致故障的情況下，一個不良行為證明可以被生成並提交給包含用戶端的區塊鏈以安全的凍結輕用戶端，並防止之後的 IBC 封包被中繼。

#### 合法性判定式

合法性判定式是由一種用戶端類型定義的一個不透明函數，用來根據當前`ConsensusState`來驗證 `Header` 。使用合法性判定式應該比通過父`Header` 和一系列網路消息進行完全共識算法重放的計算效率高很多。

合法性判定式和用戶端狀態更新邏輯是合併在一個單獨的 `checkValidityAndUpdateState`類型中的，它的定義如下：

```typescript
type checkValidityAndUpdateState = (Header) => Void
```

`checkValidityAndUpdateState` 在輸入區塊頭無效的情況下必須拋出一個異常。

如果給定的區塊頭有效，用戶端必須改變內部狀態來存儲當前確認的共識根，以及更新必要的簽名權威跟蹤（例如對驗證人集合的更新）以供後續的合法性判定式調用。

用戶端的合法性判定式可能是時間敏感的，因此，如果一段時間內（例如三週的解除綁定時間）都未提供區塊頭，將無法再更新用戶端。在這種情況下，可以允許一個被許可的實體，例如鏈治理系統或可信多重簽名介入以解凍凍結的用戶端並提供新的正確區塊頭。

#### 不良行為判定式

一個不良行為判定式是由一種用戶端類型定義的不透明函數，用於檢查數據是否對共識協議構成違規。可能是出現兩個擁有不同狀態根但在同一個區塊高度的簽名的區塊頭、一個包含無效狀態轉換的簽名的區塊頭或其他由共識算法定義的不良行為的證據。

不良行為判定式和用戶端狀態更新邏輯是合併在一個單獨的`checkMisbehaviourAndUpdateState`類型中的，它的定義如下：

```typescript
type checkMisbehaviourAndUpdateState = (bytes) => Void
```

`checkMisbehaviourAndUpdateState` 在給定證據無效的情況下必須拋出一個異常。

如果一個不良行為是有效的，用戶端還必須根據不良行為的性質去更改內部狀態，來標記先前認為有效的區塊高度為無效。

一旦檢測到不良行為，用戶端應該被凍結，之後未來的任何更新都不能被提交。諸如鏈治理系統或可信多重簽名之類的被許可的實體可能被允許干預以解凍凍結的用戶端並提供新的正確的區塊頭。

#### 用戶端狀態

用戶端狀態是由一種用戶端類型定義的不透明數據結構。它或將保留任意的內部狀態去追蹤已經被驗證過的狀態根和發生過的不良行為。

輕用戶端是一種不透明的表現形式——不同的共識算法可以定義不同的輕用戶端更新算法，但是輕用戶端必須對 IBC 處理程序公開一組通用的查詢函數。

```typescript
type ClientState = bytes
```

客戶類型必須定義一個方法用提供的共識狀態初始化一個用戶端狀態，並根據情況寫入到內部狀態中。

```typescript
type initialise = (consensusState: ConsensusState) => ClientState
```

客戶斷類型必須定義一種方法來獲取當前高度（最近驗證的區塊頭的高度）。

```typescript
type latestClientHeight = (
  clientState: ClientState)
  => uint64
```

#### 承諾根

`承諾根` 是根據 [ICS 23](../ics-023-vector-commitments) 由一種用戶端類型定義的不透明數據結構。它用於驗證處於特定最終高度（必須與特定承諾根相關聯）的狀態中是否包含特定鍵值對。

#### 狀態驗證

用戶端類型必須定義一系列函數去對用戶端追蹤的狀態機的內部狀態進行驗證。內部實現細節可能存在差異（例如，一個迴環用戶端可以直接讀取狀態訊息且不需要提供證明）。

##### 所需函數：

`verifyClientConsensusState` 驗證存儲在目標機器上的特定用戶端的共識狀態的證明。

```typescript
type verifyClientConsensusState = (
  clientState: ClientState,
  height: uint64,
  proof: CommitmentProof,
  clientIdentifier: Identifier,
  consensusStateHeight: uint64,
  consensusState: ConsensusState)
  => boolean
```

`verifyConnectionState` 驗證存儲在目標機器上的特定連接端的連接狀態的證明。

```typescript
type verifyConnectionState = (
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  connectionIdentifier: Identifier,
  connectionEnd: ConnectionEnd)
  => boolean
```

`verifyChannelState` 驗證在存儲在目標機器上的指定通道端，特定埠下的的通道狀態的證明。

```typescript
type verifyChannelState = (
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  channelEnd: ChannelEnd)
  => boolean
```

`verifyPacketData`驗證在指定的埠，指定的通道和指定的序號的傳出封包承諾的證明。

```typescript
type verifyPacketData = (
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  sequence: uint64,
  data: bytes)
  => boolean
```

`verifyPacketAcknowledgement` 在指定的埠，指定的通道和指定的序號的傳入封包的確認的證明。

```typescript
type verifyPacketAcknowledgement = (
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  sequence: uint64,
  acknowledgement: bytes)
  => boolean
```

`verifyPacketAcknowledgementAbsence` 驗證在指定的埠，指定的通道和指定的序號的未收到傳入封包確認的證明。

```typescript
type verifyPacketAcknowledgementAbsence = (
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  sequence: uint64)
  => boolean
```

`verifyNextSequenceRecv` 驗證在指定埠上和指定通道接收的下一個序號的證明。

```typescript
type verifyNextSequenceRecv = (
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  nextSequenceRecv: uint64)
  => boolean
```

#### 查詢介面

##### 鏈訊息查詢

假定這些查詢端點是由與特定用戶端關聯的鏈的節點通過 HTTP 或等效的 RPC API 公開的。

鏈必須定義 `queryHeader`，並由特定用戶端驗證，而且應允許按高度檢索區塊頭。假定此端點是不受信任的。

```typescript
type queryHeader = (height: uint64) => Header
```

鏈必須定義 `queryChainConsensusState`，並由特定用戶端驗證，以允許檢索當前共識狀態，該狀態可用於構建新用戶端。以這種方式使用時，返回的 `ConsensusState` 必須由查詢實體手動確認，因為它是主觀的。假定此端點是不受信任的。 `ConsensusState` 的確切性質可能因用戶端類型而異。

```typescript
type queryChainConsensusState = (height: uint64) => ConsensusState
```

請注意，按高度檢索過去的共識狀態（而不僅僅是當前的共識狀態）會很方便，但不是必需的。

`queryChainConsensusState` 還可以返回創建用戶端所需的其他數據，例如某些權益證明安全模型的“解除綁定期”。該數據也必須由查詢實體進行驗證。

##### 鏈上狀態查詢

該規範定義了一個通過標識符查詢用戶端狀態的函數。

```typescript
function queryClientState(identifier: Identifier): ClientState {
  return privateStore.get(clientStatePath(identifier))
}
```

`ClientState` 類型應該公開其最新的已驗證高度（如果需要，可以再使用 `queryConsensusState` 獲取其共識狀態）。

```typescript
type latestHeight = (state: ClientState) => uint64
```

用戶端類型應該定義以下標準化查詢函數，以允許中繼器和其他鏈下實體以標準API和鏈上狀態進行對接。

`queryConsensusState` 允許按高度檢索存儲的共識狀態。

```typescript
type queryConsensusState = (
  identifier: Identifier,
  height: uint64
) => ConsensusState
```

##### 證明的構造

每個用戶端類型都應該定義一些函數，以允許中繼器構造用戶端狀態驗證算法所需的證明。構造方法可能採用不同的形式，具體取決於用戶端類型。例如，Tendermint 用戶端的證明可能與存儲查詢的鍵值數據一起返回，而單機用戶端證明可能需要在單機上以互動式詢問的方式構造（因為需要用戶簽名消息）。這些函數可以由通過 RPC 到全節點的外部查詢以及本地計算或驗證構成。

```typescript
type queryAndProveClientConsensusState = (
  clientIdentifier: Identifier,
  height: uint64,
  prefix: CommitmentPrefix,
  consensusStateHeight: uint64) => ConsensusState, Proof

type queryAndProveConnectionState = (
  connectionIdentifier: Identifier,
  height: uint64,
  prefix: CommitmentPrefix) => ConnectionEnd, Proof

type queryAndProveChannelState = (
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  height: uint64,
  prefix: CommitmentPrefix) => ChannelEnd, Proof

type queryAndProvePacketData = (
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  height: uint64,
  prefix: CommitmentPrefix,
  sequence: uint64) => []byte, Proof

type queryAndProvePacketAcknowledgement = (
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  height: uint64,
  prefix: CommitmentPrefix,
  sequence: uint64) => []byte, Proof

type queryAndProvePacketAcknowledgementAbsence = (
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  height: uint64,
  prefix: CommitmentPrefix,
  sequence: uint64) => Proof

type queryAndProveNextSequenceRecv = (
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  height: uint64,
  prefix: CommitmentPrefix) => uint64, Proof
```

##### 實現策略

###### 迴環

一個本地機器的迴環用戶端僅需要讀取本地狀態，其必須具有訪問權限。

###### 簡單簽名

具有已知公鑰的獨立機器的用戶端檢查該本地機器發送的消息的簽名， 作為`Proof`參數提供。 `height`參數可以用作重放保護隨機數。

這種方式裡也可以使用多重簽名或門限簽名方案。

###### 代理用戶端

代理用戶端驗證的是目標機器的代理機器的證明。通過包含首先是一個代理機器上用戶端狀態的證明，然後是目標機器的子狀態相對於代理計算機上的用戶端狀態的證明。這使代理用戶端可以避免存儲和跟蹤目標機器本身的共識狀態，但是要以代理機器正確性的安全假設為代價。

###### 梅克爾狀態樹

對於具有梅克爾狀態樹的狀態機的用戶端，可以透過調用`verifyMembership`或`verifyNonMembership`來實現這些功能。使用經過驗證的存儲在`ClientState`中的梅克爾根，按照 [ICS 23](../ics-023-vector-commitments) 驗證處於特定高度的狀態中特定鍵/值對是否存在。

```typescript
type verifyMembership = (ClientState, uint64, CommitmentProof, Path, Value) => boolean
```

```typescript
type verifyNonMembership = (ClientState, uint64, CommitmentProof, Path) => boolean
```

### 子協議

IBC 處理程序必須實現以下定義的函數。

#### 標識符驗證

用戶端存儲在唯一的`Identifier`前綴下。 ICS 002 不要求以特定方式生成用戶端標識符，僅要求它們是唯一的即可。但是，如果需要，可以限制`Identifier`的空間。可能需要提供下面的驗證函數`validateClientIdentifier` 。

```typescript
type validateClientIdentifier = (id: Identifier) => boolean
```

如果沒有提供以上函數，預設的`validateClientIdentifier`會永遠返回`true` 。

##### 利用過去的狀態根

為了避免用戶端更新（更改狀態根）與握手中攜帶證明的交易或封包收據之間的競態條件，許多 IBC 處理程序允許調用方指定一個之前的狀態根作為參考，這類 IBC 處理程序必須確保它們對調用者傳入的區塊高度執行任何必要的檢查，以確保邏輯上的正確性。

#### 創建

透過調用`createClient`附帶特定的標識符和初始化共識狀態來創建一個用戶端。

```typescript
function createClient(
  id: Identifier,
  clientType: ClientType,
  consensusState: ConsensusState) {
    abortTransactionUnless(validateClientIdentifier(id))
    abortTransactionUnless(privateStore.get(clientStatePath(id)) === null)
    abortSystemUnless(provableStore.get(clientTypePath(id)) === null)
    clientType.initialise(consensusState)
    provableStore.set(clientTypePath(id), clientType)
}
```

#### 查詢

可以通過標識符查詢用戶端共識狀態和用戶端內部狀態，但是必須被查詢的特定路徑應由每種用戶端類型定義。

#### 更新

用戶端的更新是透過提交新的`Header`來完成的。`Identifier`用於指向邏輯將被更新的用戶端狀態。 當使用`ClientState`的合法性判定式和`ConsensusState`驗證新的`Header`時，用戶端必須相應的更新其內部狀態，還可能更新最終性承諾根和`ConsensusState`中的簽名授權邏輯。

如果一個用戶端無法繼續更新（例如，如果超過了信任期），則將不能通過與該用戶端關聯的連接和通道再發送任何封包，或者使在傳輸過程中的任何封包超時（因為無法再驗證目標鏈上的高度和時間戳）。必須進行手動干預才能重設用戶端狀態或將連接和通道遷移到另一個用戶端。無法安全的完全自動完成此操作，但是實施 IBC 的鏈可以選擇允許治理機制執行這些操作（甚至可能操作多簽或合約中的單個用戶端/連接/通道）。

```typescript
function updateClient(
  id: Identifier,
  header: Header) {
    clientType = provableStore.get(clientTypePath(id))
    abortTransactionUnless(clientType !== null)
    clientState = privateStore.get(clientStatePath(id))
    abortTransactionUnless(clientState !== null)
    clientType.checkValidityAndUpdateState(clientState, header)
}
```

#### 不良行為

如果用戶端檢測到不良行為的證據，則會發出警報，比如說可以使先前有效的狀態根變為無效並阻止其未來的更新。

```typescript
function submitMisbehaviourToClient(
  id: Identifier,
  evidence: bytes) {
    clientType = provableStore.get(clientTypePath(id))
    abortTransactionUnless(clientType !== null)
    clientState = privateStore.get(clientStatePath(id))
    abortTransactionUnless(clientState !== null)
    clientType.checkMisbehaviourAndUpdateState(clientState, evidence)
}
```

### 實現範例

一個合法性判定式範例是構建在運行單一運營者的共識算法的區塊鏈上的，其中有效區塊由這個運營者進行簽名。區塊鏈運行過程中運營者的簽名金鑰可以被改變。

用戶端特定的類型定義如下：

- `ConsensusState` 存儲最新的區塊高度和最新的公鑰
- `Header`包含一個區塊高度、一個新的承諾根、一個運營者的簽名以及可能還包括一個新的公鑰
- `checkValidityAndUpdateState` 檢查已經提交的區塊高度是否是單調遞增的以及簽名是否正確，並更改內部狀態
- `checkMisbehaviourAndUpdateState` 被用於檢查兩個相同區塊高度但承諾根不同的區塊頭，並更改內部狀態

```typescript
interface ClientState {
  frozen: boolean
  pastPublicKeys: Set<PublicKey>
  verifiedRoots: Map<uint64, CommitmentRoot>
}

interface ConsensusState {
  sequence: uint64
  publicKey: PublicKey
}

interface Header {
  sequence: uint64
  commitmentRoot: CommitmentRoot
  signature: Signature
  newPublicKey: Maybe<PublicKey>
}

interface Evidence {
  h1: Header
  h2: Header
}

// algorithm run by operator to commit a new block
function commit(
  commitmentRoot: CommitmentRoot,
  sequence: uint64,
  newPublicKey: Maybe<PublicKey>): Header {
    signature = privateKey.sign(commitmentRoot, sequence, newPublicKey)
    header = {sequence, commitmentRoot, signature, newPublicKey}
    return header
}

// initialisation function defined by the client type
function initialise(consensusState: ConsensusState): () {
  clientState = {
    frozen: false,
    pastPublicKeys: Set.singleton(consensusState.publicKey),
    verifiedRoots: Map.empty()
  }
  privateStore.set(identifier, clientState)
}

// validity predicate function defined by the client type
function checkValidityAndUpdateState(
  clientState: ClientState,
  header: Header) {
    abortTransactionUnless(consensusState.sequence + 1 === header.sequence)
    abortTransactionUnless(consensusState.publicKey.verify(header.signature))
    if (header.newPublicKey !== null) {
      consensusState.publicKey = header.newPublicKey
      clientState.pastPublicKeys.add(header.newPublicKey)
    }
    consensusState.sequence = header.sequence
    clientState.verifiedRoots[sequence] = header.commitmentRoot
}

function verifyClientConsensusState(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  clientIdentifier: Identifier,
  consensusState: ConsensusState) {
    path = applyPrefix(prefix, "clients/{clientIdentifier}/consensusStates/{height}")
    abortTransactionUnless(!clientState.frozen)
    return clientState.verifiedRoots[sequence].verifyMembership(path, consensusState, proof)
}

function verifyConnectionState(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  connectionIdentifier: Identifier,
  connectionEnd: ConnectionEnd) {
    path = applyPrefix(prefix, "connections/{connectionIdentifier}")
    abortTransactionUnless(!clientState.frozen)
    return clientState.verifiedRoots[sequence].verifyMembership(path, connectionEnd, proof)
}

function verifyChannelState(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  channelEnd: ChannelEnd) {
    path = applyPrefix(prefix, "ports/{portIdentifier}/channels/{channelIdentifier}")
    abortTransactionUnless(!clientState.frozen)
    return clientState.verifiedRoots[sequence].verifyMembership(path, channelEnd, proof)
}

function verifyPacketData(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  sequence: uint64,
  data: bytes) {
    path = applyPrefix(prefix, "ports/{portIdentifier}/channels/{channelIdentifier}/packets/{sequence}")
    abortTransactionUnless(!clientState.frozen)
    return clientState.verifiedRoots[sequence].verifyMembership(path, hash(data), proof)
}

function verifyPacketAcknowledgement(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  sequence: uint64,
  acknowledgement: bytes) {
    path = applyPrefix(prefix, "ports/{portIdentifier}/channels/{channelIdentifier}/acknowledgements/{sequence}")
    abortTransactionUnless(!clientState.frozen)
    return clientState.verifiedRoots[sequence].verifyMembership(path, hash(acknowledgement), proof)
}

function verifyPacketAcknowledgementAbsence(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  sequence: uint64) {
    path = applyPrefix(prefix, "ports/{portIdentifier}/channels/{channelIdentifier}/acknowledgements/{sequence}")
    abortTransactionUnless(!clientState.frozen)
    return clientState.verifiedRoots[sequence].verifyNonMembership(path, proof)
}

function verifyNextSequenceRecv(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  nextSequenceRecv: uint64) {
    path = applyPrefix(prefix, "ports/{portIdentifier}/channels/{channelIdentifier}/nextSequenceRecv")
    abortTransactionUnless(!clientState.frozen)
    return clientState.verifiedRoots[sequence].verifyMembership(path, nextSequenceRecv, proof)
}

// misbehaviour verification function defined by the client type
// any duplicate signature by a past or current key freezes the client
function checkMisbehaviourAndUpdateState(
  clientState: ClientState,
  evidence: Evidence) {
    h1 = evidence.h1
    h2 = evidence.h2
    abortTransactionUnless(clientState.pastPublicKeys.contains(h1.publicKey))
    abortTransactionUnless(h1.sequence === h2.sequence)
    abortTransactionUnless(h1.commitmentRoot !== h2.commitmentRoot || h1.publicKey !== h2.publicKey)
    abortTransactionUnless(h1.publicKey.verify(h1.signature))
    abortTransactionUnless(h2.publicKey.verify(h2.signature))
    clientState.frozen = true
}
```

### 屬性和不變數

- 客戶標識符是不可變的，先到先得。用戶端無法刪除（如果重複使用標識符，允許刪除意味著允許將來重放過去的封包）。

## 向後相容性

不適用。

## 向前相容性

只要新用戶端類型符合該介面，就可以隨意添加到 IBC 實現中。

## 範例實現

即將到來。

## 其他實現

即將到來。

## 歷史

2019年3月5日-初稿已完成並作為 PR 提交

2019年5月29日-進行了各種修訂，尤其是多個承諾根

2019年8月15日-進行大量返工以使用戶端介面更加清晰

2020年1月13日-用戶端類型分離和路徑更改的修訂

2020年1月26日-添加查詢介面

## 版權

本文中的所有內容均根據 [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0) 獲得許可。
