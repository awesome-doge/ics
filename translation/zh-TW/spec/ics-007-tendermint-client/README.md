---
ics: 7
title: Tendermint 用戶端
stage: 草案
category: IBC/TAO
kind: 實例化
implements: 2
author: Christopher Goes <cwgoes@tendermint.com>
created: 2019-12-10
modified: 2019-12-19
---

## 概要

本規範文件描述了使用 Tendermint 共識的區塊鏈用戶端（驗證算法）。

### 動機

使用 Tendermint 共識算法的各種狀態機可能希望與其他使用 IBC 的狀態機或單機進行交互。

### 定義

函數和術語如 [ICS 2](../ics-002-client-semantics) 中所定義。

`currentTimestamp`如 [ICS 24](../ics-024-host-requirements) 中所定義。

Tendermint 輕用戶端使用 ICS 8 中定義的通用梅克爾證明格式。

`hash`是一種通用的抗碰撞哈希函數，可以輕鬆的配置。

### 所需屬性

該規範必須滿足 ICS 2 中定義的用戶端介面。

#### 關於“可能被欺騙了”邏輯的注釋

“可能被欺騙了”檢測的基本思想是，它使我們更加保守，當我們知道網路上其他地方的另一個輕用戶端使用了稍微不同的更新模式時，會凍結我們的輕用戶端。因為可能已經被欺騙了，即使我們實際沒有被欺騙。

考慮三個鏈`A` ， `B`和`C`的拓撲，以及`A_1`和`A_2`兩個鏈`A`的用戶端，它們分別在鏈`B`和`C`上運行。依次發生以下事件：

- 鏈`A`在高度`h_0` 處生成一個塊（正確）。
- 用戶端`A_1`和`A_2`被更新到高度為`h_0`的塊。
- 鏈`A`在高度`h_0 + n` 生成一個塊（正確）。
- 用戶端`A_1`已更新到高度為`h_0 + n`的塊（用戶端`A_2`尚未更新）。
- 鏈`A`生成了第二個 （矛盾的） 高度為`h_0 + k`的區塊，並且`k <= n`。

*如果沒有* “可能被欺騙了”，則用戶端`A_2`會凍結（因為在高度`h_0 + k`處有兩個有效塊，它們比`A_2`的最新的區塊頭要新），但是*無法*凍結`A_1` ，因為`A_1`已經超過了`h_0 + k` 。

可以說，這是不利的，因為`A_1`只是“幸運”的被更新了，而`A_2`沒有，並且明顯一些拜占庭式的錯誤已經發生，應該由人或治理體系來干預處理。 “可能被欺騙了”的想法是透過讓`A_1`從可配置的過去區塊頭開始以檢測不良行為來偵測此類錯誤（因此，在這種情況下， `A_1`若能夠從`h_0`開始檢測，那麼也將被凍結 ）。

這有一個靈活的參數，即`A_1`希望從多久前開始檢查（當已更新到`h_0 + n`，`n`會是多大時，`A_1`仍然會願意查找`h_0` ）？還存在一個反作用的擔憂，即在解除綁定期之後，雙簽被認為是無成本的，我們並不想為 IBC 客戶開放一個拒絕服務攻擊的媒介。

因此，必要條件是`A_1`應該查找已存儲的最早的區塊頭，但還應對證據進行“解除期限”檢查，如果證據早於解除期限，則應避免凍結用戶端（相對於用戶端的本地時間戳）。如果擔心時鐘偏斜，可以添加一個輕微的增量。

## 技術指標

該規範依賴於[Tendermint 共識算法](https://github.com/tendermint/spec/blob/master/spec/consensus/consensus.md)和[輕用戶端算法](https://github.com/tendermint/spec/blob/master/spec/consensus/light-client.md)的正確實例化。

### 用戶端狀態

Tendermint 用戶端狀態跟蹤當前的驗證人集合，信任期，解除綁定期，最新區塊高度，最新時間戳（區塊時間）以及可能的凍結區塊高度。

```typescript
interface ClientState {
  validatorSet: List<Pair<Address, uint64>>
  trustingPeriod: uint64
  unbondingPeriod: uint64
  latestHeight: uint64
  latestTimestamp: uint64
  frozenHeight: Maybe<uint64>
}
```

### 共識狀態

Tendermint 用戶端會跟蹤所有先前已驗證的共識狀態的時間戳（區塊時間），驗證人集和和承諾根（在取消綁定期之後可以將其清除，但不應該在之前清除）。

```typescript
interface ConsensusState {
  timestamp: uint64
  validatorSet: List<Pair<Address, uint64>>
  commitmentRoot: []byte
}
```

### 區塊頭

Tendermint 用戶端頭包括區塊高度，時間戳，承諾根，完整的驗證人集合以及提交該塊的驗證人的簽名。

```typescript
interface Header {
  height: uint64
  timestamp: uint64
  commitmentRoot: []byte
  validatorSet: List<Pair<Address, uint64>>
  signatures: []Signature
}
```

### 證據

`Evidence`類型用於檢測不良行為並凍結用戶端，以防止進一步的封包流。 Tendermint 用戶端的`Evidence`包括兩個相同高度並且輕用戶端認為都是有效的區塊頭。

```typescript
interface Evidence {
  fromHeight: uint64
  h1: Header
  h2: Header
}
```

### 用戶端初始化

Tendermint 用戶端初始化要求（主觀選擇的）最新的共識狀態，包括完整的驗證人集合。

```typescript
function initialise(
  consensusState: ConsensusState, validatorSet: List<Pair<Address, uint64>>,
  height: uint64, trustingPeriod: uint64, unbondingPeriod: uint64): ClientState {
    assert(trustingPeriod < unbondingPeriod)
    assert(height > 0)
    set("clients/{identifier}/consensusStates/{height}", consensusState)
    return ClientState{
      validatorSet,
      latestHeight: height,
      latestTimestamp: consensusState.timestamp,
      trustingPeriod,
      unbondingPeriod,
      frozenHeight: null
    }
}
```

Tendermint 用戶端的`latestClientHeight`函數返回最新存儲的高度，該高度在每次驗證了新的（較新的）區塊頭時都會更新。

```typescript
function latestClientHeight(clientState: ClientState): uint64 {
  return clientState.latestHeight
}
```

### 合法性判定式

Tendermint 用戶端合法性檢查使用[Tendermint 規範中](https://github.com/tendermint/spec/tree/master/spec/consensus/light-client)描述的二分算法。如果提供的區塊頭有效，那麼將更新用戶端狀態並將新驗證的承諾寫入存儲。

```typescript
function checkValidityAndUpdateState(
  clientState: ClientState,
  header: Header) {
    // assert trusting period has not yet passed. This should fatally terminate a connection.
    assert(currentTimestamp() - clientState.latestTimestamp < clientState.trustingPeriod)
    // assert header timestamp is less than trust period in the future. This should be resolved with an intermediate header.
    assert(header.timestamp - clientState.latestTimeStamp < trustingPeriod)
    // assert header timestamp is past current timestamp
    assert(header.timestamp > clientState.latestTimestamp)
    // assert header height is newer than any we know
    assert(header.height > clientState.latestHeight)
    // call the `verify` function
    assert(verify(clientState.validatorSet, clientState.latestHeight, header))
    // update validator set
    clientState.validatorSet = header.validatorSet
    // update latest height
    clientState.latestHeight = header.height
    // update latest timestamp
    clientState.latestTimestamp = header.timestamp
    // create recorded consensus state, save it
    consensusState = ConsensusState{header.validatorSet, header.commitmentRoot, header.timestamp}
    set("clients/{identifier}/consensusStates/{header.height}", consensusState)
    // save the client
    set("clients/{identifier}", clientState)
}
```

### 不良行為判定式

Tendermint 用戶端的不良行為檢查決定於在相同高度的兩個衝突區塊頭是否都會通過輕用戶端的驗證。

```typescript
function checkMisbehaviourAndUpdateState(
  clientState: ClientState,
  evidence: Evidence) {
    // assert that the heights are the same
    assert(evidence.h1.height === evidence.h2.height)
    // assert that the commitments are different
    assert(evidence.h1.commitmentRoot !== evidence.h2.commitmentRoot)
    // fetch the previously verified commitment root & validator set
    consensusState = get("clients/{identifier}/consensusStates/{evidence.fromHeight}")
    // assert that the timestamp is not from more than an unbonding period ago
    assert(currentTimestamp() - evidence.timestamp < clientState.unbondingPeriod)
    // check if the light client "would have been fooled"
    assert(
      verify(consensusState.validatorSet, evidence.fromHeight, evidence.h1) &&
      verify(consensusState.validatorSet, evidence.fromHeight, evidence.h2)
      )
    // set the frozen height
    clientState.frozenHeight = min(clientState.frozenHeight, evidence.h1.height) // which is same as h2.height
    // save the client
    set("clients/{identifier}", clientState)
}
```

### 狀態驗證函數

Tendermint 用戶端狀態驗證函數對照先前已驗證的承諾根檢查梅克爾證明。

```typescript
function verifyClientConsensusState(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  clientIdentifier: Identifier,
  consensusStateHeight: uint64,
  consensusState: ConsensusState) {
    path = applyPrefix(prefix, "clients/{clientIdentifier}/consensusState/{consensusStateHeight}")
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{identifier}/consensusStates/{height}")
    // verify that the provided consensus state has been stored
    assert(root.verifyMembership(path, consensusState, proof))
}

function verifyConnectionState(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  connectionIdentifier: Identifier,
  connectionEnd: ConnectionEnd) {
    path = applyPrefix(prefix, "connections/{connectionIdentifier}")
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{identifier}/consensusStates/{height}")
    // verify that the provided connection end has been stored
    assert(root.verifyMembership(path, connectionEnd, proof))
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
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{identifier}/consensusStates/{height}")
    // verify that the provided channel end has been stored
    assert(root.verifyMembership(path, channelEnd, proof))
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
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{identifier}/consensusStates/{height}")
    // verify that the provided commitment has been stored
    assert(root.verifyMembership(path, hash(data), proof))
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
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{identifier}/consensusStates/{height}")
    // verify that the provided acknowledgement has been stored
    assert(root.verifyMembership(path, hash(acknowledgement), proof))
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
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{identifier}/consensusStates/{height}")
    // verify that no acknowledgement has been stored
    assert(root.verifyNonMembership(path, proof))
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
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{identifier}/consensusStates/{height}")
    // verify that the nextSequenceRecv is as claimed
    assert(root.verifyMembership(path, nextSequenceRecv, proof))
}
```

### 屬性和不變數

正確性保證和 Tendermint 輕用戶端算法相同。

## 向後相容性

不適用。

## 向前相容性

不適用。更改用戶端驗證算法將需要新的用戶端標準。

## 範例實現

還沒有。

## 其他實現

目前沒有。

## 歷史

2019年12月10日-初始版本 2019年12月19日-最終初稿

## 版權

本文中的所有內容均根據 [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0) 獲得許可。
