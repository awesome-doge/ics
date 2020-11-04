---
ics: 3
title: 連接語義
stage: 草案
category: IBC/TAO
kind: 實例化
requires: 2, 24
required-by: 4, 25
author: Christopher Goes <cwgoes@tendermint.com>, Juwoon Yun <joon@tendermint.com>
created: 2019-03-07
modified: 2019-08-25
---

## 概要

這個標準文件對 IBC *連接*的抽象進行描述：兩條獨立鏈上的兩個有狀態的對象（*連接端* ），彼此與另一條鏈上的輕用戶端關聯，並共同來促進跨鏈子狀態的驗證和封包的關聯（通過通道）。本規範描述了用於在兩條鏈上安全的建立連接的協議。

### 動機

核心 IBC 協議對封包提供了*身份認證*和*排序*語義：確保對各自來說，封包在發送鏈上被提交（根據狀態轉換的執行，例如通證託管），並且封包被有且僅有一次的按特定的順序提交和有且僅有一次的被傳遞到接收鏈。本標準中的*連接*抽象與 [ICS 2](../ics-002-client-semantics) 中定義的*用戶端*抽象一同定義了 IBC 的*身份認證*語義。排序語義在 [ICS 4](../ics-004-channel-and-packet-semantics) 中進行了描述。

### 定義

用戶端相關的類型和函數被定義在 [ICS 2](../ics-002-client-semantics) 中。

加密承諾證明相關的類型和函數被定義在 [ICS 23](../ics-023-vector-commitments) 中。

`Identifier`和其他主機狀態機的要求如 [ICS 24](../ics-024-host-requirements) 所示。標識符不一定需要是人類可讀的名稱（基本上也不應該是，來防止對標識符的搶註或爭奪）。

開放式握手協議允許每個鏈驗證用於引用另一個鏈上的連接的標識符，從而使每個鏈上的模組可以使用另一個鏈上的引用。

本規範中提到的*參與者*是能夠執行數據報的實體，並為計算/存儲付費（通過 gas 或類似的機制），但是是不被信任的。 可能的參與者包括：

- 使用帳戶金鑰簽名的最終用戶
- 自主或響應另一筆交易的鏈上智慧合約
- 響應其他交易或按計劃方式運行的鏈上模組

### 所需屬性

- 區塊鏈實現應該安全的允許不受信的參與者建立或更新連接。

#### 連接建立前階段

在建立連接之前：

- 連接階段之後的 IBC 子協議不應該能被操作，因為跨鏈子狀態還沒被驗證。
- 發起方（創建連接方）必須能夠為被連接的鏈和連接的鏈指定初始共識狀態（隱式的，例如通過發送交易）。

#### 握手期間

一旦握手協商開始：

- 只有相關的握手數據報才可以按順序被執行。
- 沒有第三條鏈可以偽裝成正在發生握手的兩條鏈中的一條

#### 完成握手後階段

一旦握手協商完成：

- 在兩個鏈上創建的連接對象均包含發起方指定的共識狀態。
- 其他連接對象不能透過重放數據報的方式在其他鏈上惡意的被創建。

## 技術指標

### 數據結構

此 ICS 定義了`ConnectionState`和`ConnectionEnd`類型：

```typescript
enum ConnectionState {
  INIT,
  TRYOPEN,
  OPEN,
}
```

```typescript
interface ConnectionEnd {
  state: ConnectionState
  counterpartyConnectionIdentifier: Identifier
  counterpartyPrefix: CommitmentPrefix
  clientIdentifier: Identifier
  counterpartyClientIdentifier: Identifier
  version: string | []string
}
```

- `state`欄位描述連接端的當前狀態。
- `counterpartyConnectionIdentifier`欄位標識與此連接關聯的對方鏈上的連接端。
- `counterpartyPrefix`欄位包含用於與此連接關聯的對方鏈上的狀態驗證的前綴。鏈應該公開一個端點，以允許中繼器查詢連接前綴。如果沒有指定，默認`counterpartyPrefix`的`"ibc"`應該被使用。
- `clientIdentifier`欄位標識與此連接關聯的用戶端。
- `counterpartyClientIdentifier`欄位標識與此連接關聯的對方鏈上的用戶端。
- `version`欄位是不透明的字串，可用於確定使用此連接的通道或封包的編碼或協議。如果未指定，則應使用默認`version` `""` 。

### 儲存路徑

連接路徑存儲在唯一標識符下。

```typescript
function connectionPath(id: Identifier): Path {
    return "connections/{id}"
}
```

從用戶端到一組連接（用於使用用戶端查找所有連接）的反向映射存儲在每個用戶端的唯一前綴下：

```typescript
function clientConnectionsPath(clientIdentifier: Identifier): Path {
    return "clients/{clientIdentifier}/connections"
}
```

### 輔助函數

`addConnectionToClient`用於將連接標識符添加到與用戶端關聯的連接集合。

```typescript
function addConnectionToClient(
  clientIdentifier: Identifier,
  connectionIdentifier: Identifier) {
    conns = privateStore.get(clientConnectionsPath(clientIdentifier))
    conns.add(connectionIdentifier)
    privateStore.set(clientConnectionsPath(clientIdentifier), conns)
}
```

`removeConnectionFromClient`用於從與用戶端關聯的連接集合中刪除某個連接標識符。

```typescript
function removeConnectionFromClient(
  clientIdentifier: Identifier,
  connectionIdentifier: Identifier) {
    conns = privateStore.get(clientConnectionsPath(clientIdentifier))
    conns.remove(connectionIdentifier)
    privateStore.set(clientConnectionsPath(clientIdentifier), conns)
}
```

輔助函數由連接所定義，以將與連接關聯的`CommitmentPrefix`傳遞給用戶端提供的驗證函數。 在規範的其他部分，這些功能必須用於檢視其他鏈的狀態，而不是直接在用戶端上調用驗證函數。

```typescript
function verifyClientConsensusState(
  connection: ConnectionEnd,
  height: uint64,
  proof: CommitmentProof,
  clientIdentifier: Identifier,
  consensusStateHeight: uint64,
  consensusState: ConsensusState) {
    client = queryClient(connection.clientIdentifier)
    return client.verifyClientConsensusState(connection, height, connection.counterpartyPrefix, proof, clientIdentifier, consensusStateHeight, consensusState)
}

function verifyConnectionState(
  connection: ConnectionEnd,
  height: uint64,
  proof: CommitmentProof,
  connectionIdentifier: Identifier,
  connectionEnd: ConnectionEnd) {
    client = queryClient(connection.clientIdentifier)
    return client.verifyConnectionState(connection, height, connection.counterpartyPrefix, proof, connectionIdentifier, connectionEnd)
}

function verifyChannelState(
  connection: ConnectionEnd,
  height: uint64,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  channelEnd: ChannelEnd) {
    client = queryClient(connection.clientIdentifier)
    return client.verifyChannelState(connection, height, connection.counterpartyPrefix, proof, portIdentifier, channelIdentifier, channelEnd)
}

function verifyPacketData(
  connection: ConnectionEnd,
  height: uint64,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  sequence: uint64,
  data: bytes) {
    client = queryClient(connection.clientIdentifier)
    return client.verifyPacketData(connection, height, connection.counterpartyPrefix, proof, portIdentifier, channelIdentifier, data)
}

function verifyPacketAcknowledgement(
  connection: ConnectionEnd,
  height: uint64,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  sequence: uint64,
  acknowledgement: bytes) {
    client = queryClient(connection.clientIdentifier)
    return client.verifyPacketAcknowledgement(connection, height, connection.counterpartyPrefix, proof, portIdentifier, channelIdentifier, acknowledgement)
}

function verifyPacketAcknowledgementAbsence(
  connection: ConnectionEnd,
  height: uint64,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  sequence: uint64) {
    client = queryClient(connection.clientIdentifier)
    return client.verifyPacketAcknowledgementAbsence(connection, height, connection.counterpartyPrefix, proof, portIdentifier, channelIdentifier)
}

function verifyNextSequenceRecv(
  connection: ConnectionEnd,
  height: uint64,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  nextSequenceRecv: uint64) {
    client = queryClient(connection.clientIdentifier)
    return client.verifyNextSequenceRecv(connection, height, connection.counterpartyPrefix, proof, portIdentifier, channelIdentifier, nextSequenceRecv)
}

function getTimestampAtHeight(
  connection: ConnectionEnd,
  height: uint64) {
    client = queryClient(connection.clientIdentifier)
    return client.queryConsensusState(height).getTimestamp()
}
```

### 子協議

本 ICS 定義了建立握手子協議。一旦握手建立，連接將不能被關閉，標識符也無法被重新分配（這防止了封包重放或者身份認證混亂）。

區塊頭追蹤和不良行為檢測在 [ICS 2](../ics-002-client-semantics) 中被定義。

![State Machine Diagram](state.png)

#### 標識符驗證

連接存儲在唯一的`Identifier`前綴下。 可以提供驗證函數`validateConnectionIdentifier`。

```typescript
type validateConnectionIdentifier = (id: Identifier) => boolean
```

如果未提供，預設的`validateConnectionIdentifier`函數將始終返回`true`。

#### 版本控制

在握手過程中，連接的兩端需要對連接關聯的版本位元組串達成一致。目前，版本位元組串的內容對於 IBC 核心協議是不透明的。將來，它可能被用於指示哪些類型的通道可以使用特定的連接，或者通道相關的數據報將使用哪種編碼格式。目前，主機狀態機可以利用版本數據來協商與 IBC 之上的自訂邏輯有關的編碼、優先度或特定與連接的元數據。

主機狀態機還可以安全的忽略版本數據或指定一個空字串。

該標準的一個實現必須定義一個函數`getCompatibleVersions` ，該函數返回它支持的版本列表，按優先度降序排列。

```typescript
type getCompatibleVersions = () => []string
```

實現必須定義一個函數 `pickVersion` 來從對方提議的版本列表中選擇一個版本。

```typescript
type pickVersion = ([]string) => string
```

#### 建立握手

建立握手子協議用於在兩條鏈上初始化彼此的共識狀態。

建立握手定義了四種數據報： *ConnOpenInit* ， *ConnOpenTry* ， *ConnOpenAck*和*ConnOpenConfirm* 。

一個正確的協議執行流程如下：（注意所有的請求都是按照 ICS 25 來制定的）

發起人 | 數據報 | 作用鏈 | 之前狀態（A，B） | 之後狀態（A，B）
--- | --- | --- | --- | ---
參與者 | `ConnOpenInit` | A | (none, none) | （INIT，none）
中繼器 | `ConnOpenTry` | B | （INIT，none） | （INIT，TRYOPEN）
中繼器 | `ConnOpenAck` | A | （INIT，TRYOPEN） | (OPEN, TRYOPEN)
中繼器 | `ConnOpenConfirm` | B | (OPEN, TRYOPEN) | (OPEN, OPEN)

在實現子協議的兩個鏈之間的建立握手結束時，具有以下屬性：

- 每條鏈都具有源自發起方所指定的對方鏈正確共識狀態。
- 每條鏈都知道且認同另一鏈上的標識符。

該子協議不需要經過許可，除了考慮反垃圾訊息。

*ConnOpenInit* 初始化鏈 A 上的連接嘗試。

```typescript
function connOpenInit(
  identifier: Identifier,
  desiredCounterpartyConnectionIdentifier: Identifier,
  counterpartyPrefix: CommitmentPrefix,
  clientIdentifier: Identifier,
  counterpartyClientIdentifier: Identifier) {
    abortTransactionUnless(validateConnectionIdentifier(identifier))
    abortTransactionUnless(provableStore.get(connectionPath(identifier)) == null)
    state = INIT
    connection = ConnectionEnd{state, desiredCounterpartyConnectionIdentifier, counterpartyPrefix,
      clientIdentifier, counterpartyClientIdentifier, getCompatibleVersions()}
    provableStore.set(connectionPath(identifier), connection)
    addConnectionToClient(clientIdentifier, identifier)
}
```

*ConnOpenTry* 中繼鏈 A 到鏈 B 的連接嘗試的通知（此代碼在鏈 B 上執行）。

```typescript
function connOpenTry(
  desiredIdentifier: Identifier,
  counterpartyConnectionIdentifier: Identifier,
  counterpartyPrefix: CommitmentPrefix,
  counterpartyClientIdentifier: Identifier,
  clientIdentifier: Identifier,
  counterpartyVersions: string[],
  proofInit: CommitmentProof,
  proofConsensus: CommitmentProof,
  proofHeight: uint64,
  consensusHeight: uint64) {
    abortTransactionUnless(validateConnectionIdentifier(desiredIdentifier))
    abortTransactionUnless(consensusHeight <= getCurrentHeight())
    expectedConsensusState = getConsensusState(consensusHeight)
    expected = ConnectionEnd{INIT, desiredIdentifier, getCommitmentPrefix(), counterpartyClientIdentifier,
                             clientIdentifier, counterpartyVersions}
    version = pickVersion(counterpartyVersions)
    connection = ConnectionEnd{TRYOPEN, counterpartyConnectionIdentifier, counterpartyPrefix,
                               clientIdentifier, counterpartyClientIdentifier, version}
    abortTransactionUnless(connection.verifyConnectionState(proofHeight, proofInit, counterpartyConnectionIdentifier, expected))
    abortTransactionUnless(connection.verifyClientConsensusState(
      proofHeight, proofConsensus, counterpartyClientIdentifier, consensusHeight, expectedConsensusState))
    previous = provableStore.get(connectionPath(desiredIdentifier))
    abortTransactionUnless(
      (previous === null) ||
      (previous.state === INIT &&
        previous.counterpartyConnectionIdentifier === counterpartyConnectionIdentifier &&
        previous.counterpartyPrefix === counterpartyPrefix &&
        previous.clientIdentifier === clientIdentifier &&
        previous.counterpartyClientIdentifier === counterpartyClientIdentifier &&
        previous.version === version))
    identifier = desiredIdentifier
    provableStore.set(connectionPath(identifier), connection)
    addConnectionToClient(clientIdentifier, identifier)
}
```

*ConnOpenAck* 對從鏈 B 返回鏈 A 的連接建立嘗試的確認消息進行中繼（此代碼在鏈 A 上執行）。

```typescript
function connOpenAck(
  identifier: Identifier,
  version: string,
  proofTry: CommitmentProof,
  proofConsensus: CommitmentProof,
  proofHeight: uint64,
  consensusHeight: uint64) {
    abortTransactionUnless(consensusHeight <= getCurrentHeight())
    connection = provableStore.get(connectionPath(identifier))
    abortTransactionUnless(connection.state === INIT || connection.state === TRYOPEN)
    expectedConsensusState = getConsensusState(consensusHeight)
    expected = ConnectionEnd{TRYOPEN, identifier, getCommitmentPrefix(),
                             connection.counterpartyClientIdentifier, connection.clientIdentifier,
                             version}
    abortTransactionUnless(connection.verifyConnectionState(proofHeight, proofTry, connection.counterpartyConnectionIdentifier, expected))
    abortTransactionUnless(connection.verifyClientConsensusState(
      proofHeight, proofConsensus, connection.counterpartyClientIdentifier, consensusHeight, expectedConsensusState))
    connection.state = OPEN
    abortTransactionUnless(getCompatibleVersions().indexOf(version) !== -1)
    connection.version = version
    provableStore.set(connectionPath(identifier), connection)
}
```

*ConnOpenConfirm* 在兩條鏈上都建立連接後確認鏈 A 與鏈 B 的連接的建立（此代碼在鏈 B 上執行）。

```typescript
function connOpenConfirm(
  identifier: Identifier,
  proofAck: CommitmentProof,
  proofHeight: uint64) {
    connection = provableStore.get(connectionPath(identifier))
    abortTransactionUnless(connection.state === TRYOPEN)
    expected = ConnectionEnd{OPEN, identifier, getCommitmentPrefix(), connection.counterpartyClientIdentifier,
                             connection.clientIdentifier, connection.version}
    abortTransactionUnless(connection.verifyConnectionState(proofHeight, proofAck, connection.counterpartyConnectionIdentifier, expected))
    connection.state = OPEN
    provableStore.set(connectionPath(identifier), connection)
}
```

#### 查詢

可以使用標識符和`queryConnection`來查詢連接。

```typescript
function queryConnection(id: Identifier): ConnectionEnd | void {
    return provableStore.get(connectionPath(id))
}
```

可以使用用戶端標識符和`queryClientConnections`來查詢與特定用戶端關聯的連接。

```typescript
function queryClientConnections(id: Identifier): Set<Identifier> {
    return privateStore.get(clientConnectionsPath(id))
}
```

### 屬性和不變性

- 連接標識符是“先到先得”的：一旦連接被商定，兩個鏈之間就會存在一對唯一的標識符。
- 連接握手不能被另一條鏈的 IBC 處理程序作為中間人來進行干預。

## 向後相容性

不適用。

## 向前相容性

此 ICS 的未來版本將在建立握手中包括版本協商。建立連接並協商版本後，可以根據 ICS 6 協商將來的版本更新。

只能在建立連接時選擇的共識協議定義的`updateConsensusState`函數允許的情況下更新共識狀態。

## 範例實現

即將到來。

## 其他實現

即將到來。

## 歷史

本文件的某些部分受[以前的 IBC 規範](https://github.com/cosmos/cosmos-sdk/tree/master/docs/spec/ibc)的啟發。

2019年3月29日-提交初稿

2019年5月17日-草稿定稿

2019年7月29日-修訂版本以跟蹤與用戶端關聯的連接集

## 版權

本文中的所有內容均根據 [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0) 獲得許可。
