---
ics: 9
title: 迴環用戶端
stage: 草案
category: IBC/TAO
kind: 實例化
author: Christopher Goes <cwgoes@tendermint.com>
created: 2020-01-17
modified: 2020-01-17
requires: 2
implements: 2
---

## 概要

本規範描述了一種迴環用戶端，該用戶端旨在通過 IBC 介面與同一帳本中存在的模組進行交互。

### 動機

如果調用模組不了解目標模組的確切位置，並且希望使用統一的 IBC 消息傳遞介面（類似於 TCP/IP 中的 `127.0.0.1` ），則迴環用戶端可能很有用。

### 定義

函數和術語如 [ICS 2](../ics-002-client-semantics) 中所定義。

### 所需屬性

應保留預期的用戶端語義，而且迴環抽象的成本應可忽略不計。

## 技術指標

### 數據結構

迴環用戶端不需要用戶端狀態，共識狀態，區塊頭或證據數據結構。

```typescript
type ClientState object

type ConsensusState object

type Header object

type Evidence object
```

### 用戶端初始化

迴環用戶端不需要初始化。將返回一個空狀態。

```typescript
function initialise(): ClientState {
  return {}
}
```

### 合法性判定式

在迴環用戶端中，無需進行合法性檢查；該函數永遠不應該被調用。

```typescript
function checkValidityAndUpdateState(
  clientState: ClientState,
  header: Header) {
    assert(false)
}
```

### 不良行為判定式

在迴環用戶端中無需進行任何不良行為檢查；該函數永遠不應該被調用。

```typescript
function checkMisbehaviourAndUpdateState(
  clientState: ClientState,
  evidence: Evidence) {
    return
}
```

### 狀態驗證函數

迴環用戶端狀態驗證函數僅讀取本地狀態。請注意，他們將需要（只讀）訪問用戶端前綴之外的鍵。

```typescript
function verifyClientConsensusState(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  clientIdentifier: Identifier,
  consensusState: ConsensusState) {
    path = applyPrefix(prefix, "consensusStates/{clientIdentifier}")
    assert(get(path) === consensusState)
}

function verifyConnectionState(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  connectionIdentifier: Identifier,
  connectionEnd: ConnectionEnd) {
    path = applyPrefix(prefix, "connection/{connectionIdentifier}")
    assert(get(path) === connectionEnd)
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
    assert(get(path) === channelEnd)
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
    assert(get(path) === commit(data))
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
    assert(get(path) === acknowledgement)
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
    assert(get(path) === nil)
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
    assert(get(path) === nextSequenceRecv)
}
```

### 屬性和不變數

語義上類似一個本地帳本的遠程用戶端。

## 向後相容性

不適用。

## 向前相容性

不適用。更改用戶端算法將需要新的用戶端標準。

## 範例實現

即將到來。

## 其他實現

目前沒有。

## 歷史

2020-01-17-初始版本

## 版權

本文中的所有內容均根據 [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0) 獲得許可。
