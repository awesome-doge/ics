---
ics: 26
title: 路由模組
stage: 草案
category: IBC/TAO
kind: 實例化
author: Christopher Goes <cwgoes@tendermint.com>
created: 2019-06-09
modified: 2019-08-25
---

## 概要

路由模組是一個輔助模組的默認實現，該模組將接受外部數據報並調用區塊鏈間通信協議處理程序來處理握手和封包中繼。 路由模組維護一個模組的查找表，當收到封包時，該表可用於查找和調用模組，因此外部中繼器僅需要將封包中繼到路由模組。

### 動機

默認的 IBC 處理程序使用接收方調用模式，其中模組必須單獨調用 IBC 處理程序才能綁定到埠，啟動握手，接受握手，發送和接收封包等。這是靈活而簡單的（請參閱[設計模式](../../ibc/5_IBC_DESIGN_PATTERNS.md) ）。 但是理解起來有些棘手，中繼器進程可能需要額外的工作，中繼器進程必須跟蹤多個模組的狀態。該標準描述了一個 IBC“路由模組”，以自動執行大部分常用功能，路由封包並簡化中繼器的任務。

路由模組還可以扮演 [ICS 5](../ics-005-port-allocation) 中討論的模組管理器的角色，並實現確定何時允許模組綁定到埠以及可以命名哪些埠的邏輯。

### 定義

IBC 處理程序接口提供的所有函數均在 [ICS 25](../ics-025-handler-interface) 中定義。

函數 `newCapability` 和 `authenticateCapability` 在 [ICS 5](../ics-005-port-allocation) 中定義。

### 所需屬性

- 模組應該能夠通過路由模組綁定到埠和獲得通道。
- 除了調用中間層外，不應為封包發送和接收增加任何開銷。
- 當路由模組需要對封包操作時，路由模組應在模組上調用指定的處理程序函數。

## 技術指標

> 注意：如果主機狀態機正在使用對象能力認證（請參閱 [ICS 005](../ics-005-port-allocation) ），則所有使用埠的函數都需要帶有一個附加的能力參數。

### 模組回調介面

模組必須向路由模組暴露以下函數簽名，這些簽名在收到各種數據報後即被調用：

```typescript
function onChanOpenInit(
  order: ChannelOrder,
  connectionHops: [Identifier],
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyPortIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  version: string) {
    // defined by the module
}

function onChanOpenTry(
  order: ChannelOrder,
  connectionHops: [Identifier],
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyPortIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  version: string,
  counterpartyVersion: string) {
    // defined by the module
}

function onChanOpenAck(
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  version: string) {
    // defined by the module
}

function onChanOpenConfirm(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
    // defined by the module
}

function onChanCloseInit(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
    // defined by the module
}

function onChanCloseConfirm(
  portIdentifier: Identifier,
  channelIdentifier: Identifier): void {
    // defined by the module
}

function onRecvPacket(packet: Packet): bytes {
    // defined by the module, returns acknowledgement
}

function onTimeoutPacket(packet: Packet) {
    // defined by the module
}

function onAcknowledgePacket(packet: Packet) {
    // defined by the module
}

function onTimeoutPacketClose(packet: Packet) {
    // defined by the module
}
```

如果出現失敗，必須拋出異常以拒絕握手和傳入的封包等。

它們在`ModuleCallbacks`介面中組合在一起：

```typescript
interface ModuleCallbacks {
  onChanOpenInit: onChanOpenInit,
  onChanOpenTry: onChanOpenTry,
  onChanOpenAck: onChanOpenAck,
  onChanOpenConfirm: onChanOpenConfirm,
  onChanCloseConfirm: onChanCloseConfirm
  onRecvPacket: onRecvPacket
  onTimeoutPacket: onTimeoutPacket
  onAcknowledgePacket: onAcknowledgePacket,
  onTimeoutPacketClose: onTimeoutPacketClose
}
```

當模組綁定到埠時，將提供回調。

```typescript
function callbackPath(portIdentifier: Identifier): Path {
    return "callbacks/{portIdentifier}"
}
```

還將存儲調用模組標識符以供將來更改回調時進行身份認證。

```typescript
function authenticationPath(portIdentifier: Identifier): Path {
    return "authentication/{portIdentifier}"
}
```

### 埠綁定作為模組管理器

IBC 路由模組位於處理程序模組（ [ICS 25](../ics-025-handler-interface) ）與主機狀態機上的各個模組之間。

充當模組管理器的路由模組區分兩種埠：

- “現有名稱”埠：例如具有標準化優先含義的“bank”，不應以先到先得的方式使用
- “新名稱”埠：沒有先驗關係的新身份（可能是智慧合約），新的隨機數埠，之後的埠名稱可以通過另一個通道通訊得到

當主機狀態機實例化路由模組時，會分配一組現有名稱以及相應的模組。 然後，路由模組允許模組隨時分配新埠，但是它們必須使用特定的標準化前綴。

模組可以調用函數`bindPort`以便通過路由模組綁定到埠並設置回調。

```typescript
function bindPort(
  id: Identifier,
  callbacks: Callbacks): CapabilityKey {
    abortTransactionUnless(privateStore.get(callbackPath(id)) === null)
    privateStore.set(callbackPath(id), callbacks)
    capability = handler.bindPort(id)
    claimCapability(authenticationPath(id), capability)
    return capability
}
```

模組可以調用函數`updatePort`來更改回調。

```typescript
function updatePort(
  id: Identifier,
  capability: CapabilityKey,
  newCallbacks: Callbacks) {
    abortTransactionUnless(authenticateCapability(authenticationPath(id), capability))
    privateStore.set(callbackPath(id), newCallbacks)
}
```

模組可以調用函數`releasePort`來釋放以前使用的埠。

> 警告：釋放埠將允許其他模組綁定到該埠，並可能攔截傳入的通道創建握手請求。只有在安全的情況下，模組才應釋放埠。

```typescript
function releasePort(
  id: Identifier,
  capability: CapabilityKey) {
    abortTransactionUnless(authenticateCapability(authenticationPath(id), capability))
    handler.releasePort(id)
    privateStore.delete(callbackPath(id))
    privateStore.delete(authenticationPath(id))
}
```

路由模組可以使用函數`lookupModule`查找綁定到特定埠的回調。

```typescript
function lookupModule(portId: Identifier) {
    return privateStore.get(callbackPath(portId))
}
```

### 數據報處理程序（寫）

*數據報*是路由模組做為交易接受的外部數據 Blob。本部分為每個數據報定義一個*處理函數* ， 當關聯的數據報在交易中提交給路由模組時執行。

所有數據報也可以由其他模組安全的提交給路由模組。

除了明確指出，不假定任何消息簽名或數據有效性檢查。

#### 用戶端生命週期管理

`ClientCreate`使用指定的標識符和共識狀態創建一個新的輕用戶端。

```typescript
interface ClientCreate {
  identifier: Identifier
  type: ClientType
  consensusState: ConsensusState
}
```

```typescript
function handleClientCreate(datagram: ClientCreate) {
    handler.createClient(datagram.identifier, datagram.type, datagram.consensusState)
}
```

`ClientUpdate`使用指定的標識符和新區塊頭更新現有的輕用戶端。

```typescript
interface ClientUpdate {
  identifier: Identifier
  header: Header
}
```

```typescript
function handleClientUpdate(datagram: ClientUpdate) {
    handler.updateClient(datagram.identifier, datagram.header)
}
```

`ClientSubmitMisbehaviour`使用指定的標識符向現有的輕用戶端提交不良行為證明。

```typescript
interface ClientMisbehaviour {
  identifier: Identifier
  evidence: bytes
}
```

```typescript
function handleClientMisbehaviour(datagram: ClientUpdate) {
    handler.submitMisbehaviourToClient(datagram.identifier, datagram.evidence)
}
```

#### 連接生命週期管理

`ConnOpenInit`數據報開始與另一個鏈上的 IBC 模組的連接的握手過程。

```typescript
interface ConnOpenInit {
  identifier: Identifier
  desiredCounterpartyIdentifier: Identifier
  clientIdentifier: Identifier
  counterpartyClientIdentifier: Identifier
  version: string
}
```

```typescript
function handleConnOpenInit(datagram: ConnOpenInit) {
    handler.connOpenInit(
      datagram.identifier,
      datagram.desiredCounterpartyIdentifier,
      datagram.clientIdentifier,
      datagram.counterpartyClientIdentifier,
      datagram.version
    )
}
```

`ConnOpenTry`數據報接受從另一個鏈上的 IBC 模組發來的握手請求。

```typescript
interface ConnOpenTry {
  desiredIdentifier: Identifier
  counterpartyConnectionIdentifier: Identifier
  counterpartyClientIdentifier: Identifier
  clientIdentifier: Identifier
  version: string
  counterpartyVersion: string
  proofInit: CommitmentProof
  proofConsensus: CommitmentProof
  proofHeight: uint64
  consensusHeight: uint64
}
```

```typescript
function handleConnOpenTry(datagram: ConnOpenTry) {
    handler.connOpenTry(
      datagram.desiredIdentifier,
      datagram.counterpartyConnectionIdentifier,
      datagram.counterpartyClientIdentifier,
      datagram.clientIdentifier,
      datagram.version,
      datagram.counterpartyVersion,
      datagram.proofInit,
      datagram.proofConsensus,
      datagram.proofHeight,
      datagram.consensusHeight
    )
}
```

`ConnOpenAck`數據報確認另一條鏈上的 IBC 模組接受了握手。

```typescript
interface ConnOpenAck {
  identifier: Identifier
  version: string
  proofTry: CommitmentProof
  proofConsensus: CommitmentProof
  proofHeight: uint64
  consensusHeight: uint64
}
```

```typescript
function handleConnOpenAck(datagram: ConnOpenAck) {
    handler.connOpenAck(
      datagram.identifier,
      datagram.version,
      datagram.proofTry,
      datagram.proofConsensus,
      datagram.proofHeight,
      datagram.consensusHeight
    )
}
```

`ConnOpenConfirm`數據報確認另一個鏈上的 IBC 模組的握手確認並完成連接。

```typescript
interface ConnOpenConfirm {
  identifier: Identifier
  proofAck: CommitmentProof
  proofHeight: uint64
}
```

```typescript
function handleConnOpenConfirm(datagram: ConnOpenConfirm) {
    handler.connOpenConfirm(
      datagram.identifier,
      datagram.proofAck,
      datagram.proofHeight
    )
}
```

#### 通道生命週期管理

```typescript
interface ChanOpenInit {
  order: ChannelOrder
  connectionHops: [Identifier]
  portIdentifier: Identifier
  channelIdentifier: Identifier
  counterpartyPortIdentifier: Identifier
  counterpartyChannelIdentifier: Identifier
  version: string
}
```

```typescript
function handleChanOpenInit(datagram: ChanOpenInit) {
    module = lookupModule(datagram.portIdentifier)
    module.onChanOpenInit(
      datagram.order,
      datagram.connectionHops,
      datagram.portIdentifier,
      datagram.channelIdentifier,
      datagram.counterpartyPortIdentifier,
      datagram.counterpartyChannelIdentifier,
      datagram.version
    )
    handler.chanOpenInit(
      datagram.order,
      datagram.connectionHops,
      datagram.portIdentifier,
      datagram.channelIdentifier,
      datagram.counterpartyPortIdentifier,
      datagram.counterpartyChannelIdentifier,
      datagram.version
    )
}
```

```typescript
interface ChanOpenTry {
  order: ChannelOrder
  connectionHops: [Identifier]
  portIdentifier: Identifier
  channelIdentifier: Identifier
  counterpartyPortIdentifier: Identifier
  counterpartyChannelIdentifier: Identifier
  version: string
  counterpartyVersion: string
  proofInit: CommitmentProof
  proofHeight: uint64
}
```

```typescript
function handleChanOpenTry(datagram: ChanOpenTry) {
    module = lookupModule(datagram.portIdentifier)
    module.onChanOpenTry(
      datagram.order,
      datagram.connectionHops,
      datagram.portIdentifier,
      datagram.channelIdentifier,
      datagram.counterpartyPortIdentifier,
      datagram.counterpartyChannelIdentifier,
      datagram.version,
      datagram.counterpartyVersion
    )
    handler.chanOpenTry(
      datagram.order,
      datagram.connectionHops,
      datagram.portIdentifier,
      datagram.channelIdentifier,
      datagram.counterpartyPortIdentifier,
      datagram.counterpartyChannelIdentifier,
      datagram.version,
      datagram.counterpartyVersion,
      datagram.proofInit,
      datagram.proofHeight
    )
}
```

```typescript
interface ChanOpenAck {
  portIdentifier: Identifier
  channelIdentifier: Identifier
  version: string
  proofTry: CommitmentProof
  proofHeight: uint64
}
```

```typescript
function handleChanOpenAck(datagram: ChanOpenAck) {
    module.onChanOpenAck(
      datagram.portIdentifier,
      datagram.channelIdentifier,
      datagram.version
    )
    handler.chanOpenAck(
      datagram.portIdentifier,
      datagram.channelIdentifier,
      datagram.version,
      datagram.proofTry,
      datagram.proofHeight
    )
}
```

```typescript
interface ChanOpenConfirm {
  portIdentifier: Identifier
  channelIdentifier: Identifier
  proofAck: CommitmentProof
  proofHeight: uint64
}
```

```typescript
function handleChanOpenConfirm(datagram: ChanOpenConfirm) {
    module = lookupModule(datagram.portIdentifier)
    module.onChanOpenConfirm(
      datagram.portIdentifier,
      datagram.channelIdentifier
    )
    handler.chanOpenConfirm(
      datagram.portIdentifier,
      datagram.channelIdentifier,
      datagram.proofAck,
      datagram.proofHeight
    )
}
```

```typescript
interface ChanCloseInit {
  portIdentifier: Identifier
  channelIdentifier: Identifier
}
```

```typescript
function handleChanCloseInit(datagram: ChanCloseInit) {
    module = lookupModule(datagram.portIdentifier)
    module.onChanCloseInit(
      datagram.portIdentifier,
      datagram.channelIdentifier
    )
    handler.chanCloseInit(
      datagram.portIdentifier,
      datagram.channelIdentifier
    )
}
```

```typescript
interface ChanCloseConfirm {
  portIdentifier: Identifier
  channelIdentifier: Identifier
  proofInit: CommitmentProof
  proofHeight: uint64
}
```

```typescript
function handleChanCloseConfirm(datagram: ChanCloseConfirm) {
    module = lookupModule(datagram.portIdentifier)
    module.onChanCloseConfirm(
      datagram.portIdentifier,
      datagram.channelIdentifier
    )
    handler.chanCloseConfirm(
      datagram.portIdentifier,
      datagram.channelIdentifier,
      datagram.proofInit,
      datagram.proofHeight
    )
}
```

#### 封包中繼

封包直接由模組發送（由模組調用 IBC 處理程序）。

```typescript
interface PacketRecv {
  packet: Packet
  proof: CommitmentProof
  proofHeight: uint64
}
```

```typescript
function handlePacketRecv(datagram: PacketRecv) {
    module = lookupModule(datagram.packet.sourcePort)
    acknowledgement = module.onRecvPacket(datagram.packet)
    handler.recvPacket(
      datagram.packet,
      datagram.proof,
      datagram.proofHeight,
      acknowledgement
    )
}
```

```typescript
interface PacketAcknowledgement {
  packet: Packet
  acknowledgement: string
  proof: CommitmentProof
  proofHeight: uint64
}
```

```typescript
function handlePacketAcknowledgement(datagram: PacketAcknowledgement) {
    module = lookupModule(datagram.packet.sourcePort)
    module.onAcknowledgePacket(
      datagram.packet,
      datagram.acknowledgement
    )
    handler.acknowledgePacket(
      datagram.packet,
      datagram.acknowledgement,
      datagram.proof,
      datagram.proofHeight
    )
}
```

#### 封包超時

```typescript
interface PacketTimeout {
  packet: Packet
  proof: CommitmentProof
  proofHeight: uint64
  nextSequenceRecv: Maybe<uint64>
}
```

```typescript
function handlePacketTimeout(datagram: PacketTimeout) {
    module = lookupModule(datagram.packet.sourcePort)
    module.onTimeoutPacket(datagram.packet)
    handler.timeoutPacket(
      datagram.packet,
      datagram.proof,
      datagram.proofHeight,
      datagram.nextSequenceRecv
    )
}
```

```typescript
interface PacketTimeoutOnClose {
  packet: Packet
  proof: CommitmentProof
  proofHeight: uint64
}
```

```typescript
function handlePacketTimeoutOnClose(datagram: PacketTimeoutOnClose) {
    module = lookupModule(datagram.packet.sourcePort)
    module.onTimeoutPacket(datagram.packet)
    handler.timeoutOnClose(
      datagram.packet,
      datagram.proof,
      datagram.proofHeight
    )
}
```

#### 超時關閉和封包清理

```typescript
interface PacketCleanup {
  packet: Packet
  proof: CommitmentProof
  proofHeight: uint64
  nextSequenceRecvOrAcknowledgement: Either<uint64, bytes>
}
```

```typescript
function handlePacketCleanup(datagram: PacketCleanup) {
    handler.cleanupPacket(
      datagram.packet,
      datagram.proof,
      datagram.proofHeight,
      datagram.nextSequenceRecvOrAcknowledgement
    )
}
```

### 查詢（只讀）函數

用戶端，連接和通道的所有查詢函數應直接由 IBC 處理程序模組暴露出來（只讀）。

### 介面用法範例

有關用法範例，請參見 [ICS 20](../ics-020-fungible-token-transfer) 。

### 屬性和不變數

- 代理埠綁定是先到先服務：模組通過  IBC路由模組綁定到埠後，只有該模組才能使用該埠，直到模組釋放它為止。

## 向後相容性

不適用。

## 向前相容性

路由模組與 IBC 處理程序介面緊密相關。

## 範例實現

即將到來。

## 其他實現

即將到來。

## 歷史

2019年6月9日-提交的草案

2019年7月28日-重大修訂

2019年8月25日-重大修訂

## 版權

本文中的所有內容均根據 [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0) 獲得許可。
