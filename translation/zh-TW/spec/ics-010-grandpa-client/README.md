---
ics: 10
title: GRANDPA 用戶端
stage: 草案
category: IBC/TAO
kind: 實例化
author: Yuanchao Sun <ys@cdot.network>, John Wu <john@cdot.network>
created: 2020-03-15
implements: 2
---

## 概要

本規範文件描述了使用 GRANDPA 最終性小工具的區塊鏈用戶端（驗證算法）。

GRANDPA（GHOST-based Recursive Ancestor Deriving Prefix Agreement）是 Polkadot 中繼鏈將會使用的一個最終性小工具。它現在有一個 Rust 語言實現，並且是 Substrate 框架的一部分，因此使用 Substrate 構建的區塊鏈很可能會使用 GRANDPA 作為其最終性小工具。

### 動機

使用 GRANDPA 最終性小工具的區塊鏈可能希望通過 IBC 與其他狀態機或單機進行交互。

### 定義

功能和術語如 [ICS 2](../ics-002-client-semantics) 中所定義。

### 所需屬性

該規範必須滿足 ICS 2 中定義的用戶端介面。

## 技術指標

該規範依賴於 [GRANDPA 最終性小工具](https://github.com/w3f/consensus/blob/master/pdf/grandpa.pdf)及其輕用戶端算法的正確實例化。

### 客戶狀態

GRANDPA 用戶端狀態跟蹤最新區塊高度和可能的凍結區塊高度。

```typescript
interface ClientState {
  latestHeight: uint64
  frozenHeight: Maybe<uint64>
}
```

### 權威集合

GRANDPA 的一組權威帳戶。

```typescript
interface AuthoritySet {
  // this is incremented every time the set changes
  setId: uint64
  authorities: List<Pair<AuthorityId, AuthorityWeight>>
}
```

### 共識狀態

GRANDPA 用戶端跟蹤所有先前已驗證的共識狀態的權威集合和承諾根。

```typescript
interface ConsensusState {
  authoritySet: AuthoritySet
  commitmentRoot: []byte
}
```

### 區塊頭

GRANDPA 用戶端區塊頭包括區塊高度，承諾根，塊的確定性證明和權威集合。 （實際上，區塊頭中包含的是一個權威集合的證明，而不是權威集合本身，但是我們可以使用一個固定的鍵來驗證證明並提取出真實集合，這裡忽略了細節）

```typescript
interface Header {
  height: uint64
  commitmentRoot: []byte
  justification: Justification
  authoritySet: AuthoritySet
}
```

### 確定性證明

一個 GRANDPA 的塊確定性證明，它包括一個提交訊息和一個祖先證明，其中包括所有預提交目標塊到提交目標塊之間的所有區塊頭。例如，最新的塊是 A-B-C-D-E-F，其中 A 是最後敲定的塊，F 是可以收集到多數投票的位置（投票可能在 B，C，D，E，F 上）。那麼證明需要包括從 F 到 A 的所有區塊頭。

```typescript
interface Justification {
  round: uint64
  commit: Commit
  votesAncestries: []Header
}
```

### 提交訊息

提交消息，它是已簽名的預提交的匯總。

```typescript
interface Commit {
  precommits: []SignedPrecommit
}

interface SignedPrecommit {
  targetHash: Hash
  signature: Signature
  id: AuthorityId
}
```

### 證據

`Evidence`類型用於檢測不良行為並凍結用戶端-以防止進一步的封包流-如果適用。 GRANDPA 用戶端`Evidence`由兩個高度相同的，輕用戶端認為都是有效的區塊頭組成。

```typescript
interface Evidence {
  fromHeight: uint64
  h1: Header
  h2: Header
}
```

### 客戶初始化

GRANDPA 用戶端初始化要求（主觀選擇）一個最新的共識狀態，包括完整的權威集合。

```typescript
function initialise(identifier: Identifier, height: uint64, consensusState: ConsensusState): ClientState {
    set("clients/{identifier}/consensusStates/{height}", consensusState)
    return ClientState{
      latestHeight: height,
      frozenHeight: null,
    }
}
```

GRANDPA 用戶端的`latestClientHeight`函數返回最新存儲的區塊高度，該高度在每次驗證一個新的（更接近現在的）區塊頭時都會更新。

```typescript
function latestClientHeight(clientState: ClientState): uint64 {
  return clientState.latestHeight
}
```

### 合法性判定式

GRANDPA 用戶端合法性檢查將驗證區塊頭是否由當前權威集合簽名，並驗證權威集合證明以確定是否存在對權威集合更改。如果提供的區塊頭有效，那麼將更新用戶端狀態並將新驗證的承諾寫入存儲。

```typescript
function checkValidityAndUpdateState(
  clientState: ClientState,
  header: Header) {
    // assert header height is newer than any we know
    assert(header.height > clientState.latestHeight)
    consensusState = get("clients/{identifier}/consensusStates/{clientState.latestHeight}")
    // verify that the provided header is valid
    assert(verify(consensusState.authoritySet, header))
    // update latest height
    clientState.latestHeight = header.height
    // create recorded consensus state, save it
    consensusState = ConsensusState{header.authoritySet, header.commitmentRoot}
    set("clients/{identifier}/consensusStates/{header.height}", consensusState)
    // save the client
    set("clients/{identifier}", clientState)
}

function verify(
  authoritySet: AuthoritySet,
  header: Header): boolean {
  let visitedHashes: Hash[]
  for (const signedPrecommit of Header.justification.commit.precommits) {
    if (checkSignature(authoritySet, signedPrecommit)) {
      visitedHashes.push(signedPrecommit.targetHash)
    }
  }
  return visitedHashes.equals(Header.justification.votesAncestries.map(hash))
}
```

### 不良行為判定式

GRANDPA 用戶端的不良行為檢查將確定在相同高度的兩個衝突的區塊頭是否都輕用戶端認定有效。

```typescript
function checkMisbehaviourAndUpdateState(
  clientState: ClientState,
  evidence: Evidence) {
    // assert that the heights are the same
    assert(evidence.h1.height === evidence.h2.height)
    // assert that the commitments are different
    assert(evidence.h1.commitmentRoot !== evidence.h2.commitmentRoot)
    // fetch the previously verified commitment root & authority set
    consensusState = get("clients/{identifier}/consensusStates/{evidence.fromHeight}")
    // check if the light client "would have been fooled"
    assert(
      verify(consensusState.authoritySet, evidence.h1) &&
      verify(consensusState.authoritySet, evidence.h2)
      )
    // set the frozen height
    clientState.frozenHeight = min(clientState.frozenHeight, evidence.h1.height) // which is same as h2.height
    // save the client
    set("clients/{identifier}", clientState)
}
```

### 狀態驗證函數

GRANDPA 用戶端狀態驗證函數對照先前已驗證的承諾根檢查梅克爾證明。

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

正確性保證和 GRANDPA 輕用戶端算法相同。

## 向後相容性

不適用。

## 向前相容性

不適用。更改用戶端驗證算法將需要新的用戶端標準。

## 範例實現

還沒有。

## 其他實現

目前沒有。

## 歷史

2020年3月15日-初始版本

## 版權

本文中的所有內容均根據 [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0) 獲得許可。
