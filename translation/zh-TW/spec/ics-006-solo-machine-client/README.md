---
ics: 6
title: 單機用戶端
stage: 草案
category: IBC/TAO
kind: 實例化
implements: 2
author: Christopher Goes <cwgoes@tendermint.com>
created: 2019-12-09
modified: 2019-12-09
---

## 概要

本規範文件描述了具有單個可更新公鑰的單機用戶端（驗證算法），該用戶端實現了 [ICS 2](../ics-002-client-semantics) 介面。

### 動機

單機，可能是諸如手機，瀏覽器或筆記型電腦之類的設備，它們希望與使用 IBC 的其他機器和多副本帳本進行交互，並且可以透過統一的用戶端介面來實現。

單機用戶端大致類似於“隱式帳戶”，可以用來代替帳本上的“常規交易”，從而允許所有交易通過 IBC 的統一介面進行。

### 定義

函數和術語如 [ICS 2](../ics-002-client-semantics) 中所定義。

### 所需屬性

該規範必須滿足 [ICS 2](../ics-002-client-semantics) 中定義的用戶端介面。

從概念上講，我們假設有一個“全局的大簽名表”（生成的簽名是公開的）並相應的包含了重放保護。

## 技術指標

該規範包含 [ICS 2](../ics-002-client-semantics) 定義的所有函數的實現。

### 用戶端狀態

單機的`ClientState`就是簡單的用戶端是否被凍結。

```typescript
interface ClientState {
  frozen: boolean
  consensusState: ConsensusState
}
```

### 共識狀態

單機的`ConsensusState`由當前的公鑰和序號組成。

```typescript
interface ConsensusState {
  sequence: uint64
  publicKey: PublicKey
}
```

### 區塊頭

`Header`僅在機器希望更新公鑰時才由單機提供。

```typescript
interface Header {
  sequence: uint64
  signature: Signature
  newPublicKey: PublicKey
}
```

### 證據

單機的不良行為的`Evidence`包括一個序號和該序號上不同消息的兩個簽名。

```typescript
interface SignatureAndData {
  sig: Signature
  data: []byte
}

interface Evidence {
  sequence: uint64
  signatureOne: SignatureAndData
  signatureTwo: SignatureAndData
}
```

### 用戶端初始化

單機用戶端`initialise`函數以初始共識狀態啟動一個未凍結的用戶端。

```typescript
function initialise(consensusState: ConsensusState): ClientState {
  return {
    frozen: false,
    consensusState
  }
}
```

單機用戶端`latestClientHeight`函數返回最新的序號。

```typescript
function latestClientHeight(clientState: ClientState): uint64 {
  return clientState.consensusState.sequence
}
```

### 合法性判定式

單機用戶端的`checkValidityAndUpdateState`函數檢查當前註冊的公共金鑰是否對新的公共金鑰和正確的序號進行了簽名。

```typescript
function checkValidityAndUpdateState(
  clientState: ClientState,
  header: Header) {
  assert(header.sequence === clientState.consensusState.sequence)
  assert(checkSignature(header.newPublicKey, header.sequence, header.signature))
  clientState.consensusState.publicKey = header.newPublicKey
  clientState.consensusState.sequence++
}
```

### 不良行為判定式

任何當前公鑰在不同消息上的重複簽名都會凍結單機用戶端。

```typescript
function checkMisbehaviourAndUpdateState(
  clientState: ClientState,
  evidence: Evidence) {
    h1 = evidence.h1
    h2 = evidence.h2
    pubkey = clientState.consensusState.publicKey
    assert(evidence.h1.signature.data !== evidence.h2.signature.data)
    assert(checkSignature(pubkey, evidence.sequence, evidence.h1.signature.sig))
    assert(checkSignature(pubkey, evidence.sequence, evidence.h2.signature.sig))
    clientState.frozen = true
}
```

### 狀態驗證函數

所有單機用戶端狀態驗證函數都僅檢查簽名，該簽名必須由單機提供。

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
    abortTransactionUnless(!clientState.frozen)
    value = clientState.consensusState.sequence + path + consensusState
    assert(checkSignature(clientState.consensusState.pubKey, value, proof))
    clientState.consensusState.sequence++
}

function verifyConnectionState(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  connectionIdentifier: Identifier,
  connectionEnd: ConnectionEnd) {
    path = applyPrefix(prefix, "connection/{connectionIdentifier}")
    abortTransactionUnless(!clientState.frozen)
    value = clientState.consensusState.sequence + path + connectionEnd
    assert(checkSignature(clientState.consensusState.pubKey, value, proof))
    clientState.consensusState.sequence++
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
    value = clientState.consensusState.sequence + path + channelEnd
    assert(checkSignature(clientState.consensusState.pubKey, value, proof))
    clientState.consensusState.sequence++
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
    value = clientState.consensusState.sequence + path + data
    assert(checkSignature(clientState.consensusState.pubKey, value, proof))
    clientState.consensusState.sequence++
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
    value = clientState.consensusState.sequence + path + acknowledgement
    assert(checkSignature(clientState.consensusState.pubKey, value, proof))
    clientState.consensusState.sequence++
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
    value = clientState.consensusState.sequence + path
    assert(checkSignature(clientState.consensusState.pubKey, value, proof))
    clientState.consensusState.sequence++
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
    value = clientState.consensusState.sequence + path + nextSequenceRecv
    assert(checkSignature(clientState.consensusState.pubKey, value, proof))
    clientState.consensusState.sequence++
}
```

### 屬性和不變數

實例化 [ICS 2](../ics-002-client-semantics) 中定義的介面。

## 向後相容性

不適用。

## 向前相容性

不適用。更改用戶端驗證算法將需要新的用戶端標準。

## 範例實現

還沒有。

## 其他實現

目前沒有。

## 歷史

2019年12月9日-初始版本 2019年12月17日-最終初稿

## 版權

本文中的所有內容均根據 [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0) 獲得許可。
