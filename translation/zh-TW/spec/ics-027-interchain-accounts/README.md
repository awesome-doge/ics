---
ics: 27
title: 鏈間帳戶
stage: 草案
category: IBC/TAO
requires: 25, 26
kind: 實例化
author: Tony Yun <yunjh1994@everett.zone>, Dogemos <josh@tendermint.com>
created: 2019-08-01
modified: 2019-12-02
---

## 概要

該標準文件指定了不同鏈之間 IBC 通道之上的帳戶管理系統的封包數據結構，狀態機處理邏輯和編碼詳細訊息。

### 動機

在以太坊上，有兩種類型的帳戶：受私鑰控制的外部擁有帳戶和受其合約代碼（ [ref](https://github.com/ethereum/wiki/wiki/White-Paper) ）控制的合約帳戶。與以太坊的 CA（合約帳戶）相似，鏈間帳戶由另一條鏈管理，同時保留普通帳戶的所有功能（例如，質押，發送，投票等）。以太坊 CA 的合約邏輯是在以太坊的 EVM 中執行的，而鏈間帳戶由另一條鏈通過 IBC 進行管理，從而使帳戶所有者可以完全控制其行為。

### 定義

IBC 處理程序介面和 IBC 路由模組介面分別在 [ICS 25](../ics-025-handler-interface) 和 [ICS 26](../ics-026-routing-module) 中所定義。

### 所需屬性

- 無需許可
- 故障容忍：鏈間帳戶必須遵循其宿主鏈的規則，即使在對方鏈（管理帳戶的鏈）出現拜占庭行為時也是如此
- 控制帳戶的鏈必須按照鏈的邏輯非同步處理結果。如果交易成功，則結果應為 0x0，如果交易失敗，則結果應為 0x0 以外的錯誤代碼。
- 發送和接收交易將在有序通道中進行處理，在該通道中，封包將按照其發送的順序傳遞。

## 技術指標

鏈間帳戶的實現是不對稱的。這意味著每個鏈可以具有不同的方式來生成鏈間帳戶並反序列化交易位元組和它們可以執行的不同交易集。例如，使用 Cosmos SDK 的鏈將使用 Amino 對 tx 位元組進行反序列化，但是如果對方鏈是以太坊上的智慧合約，則它可能會通過 ABI 對 tx 位元組進行反序列化，這是智慧合約的最小序列化算法。 鏈間帳戶規範定義了註冊鏈間帳戶和傳輸 tx 位元組的一般方法。對方鏈負責反序列化和執行 tx 位元組，並且發送鏈應事先知道對方鏈將如何處理 tx 位元組。

每個鏈必須滿足以下功能才能創建鏈間帳戶：

- 新的鏈間帳戶不得與現有帳戶衝突。
- 每個鏈必須跟蹤是哪個對方鏈創建了新的鏈間帳戶。

同樣，每個鏈必須知道對方鏈如何序列化/反序列化交易位元組，以便通過 IBC 發送交易。對方鏈必須通過驗證交易簽名人的權限來安全執行 IBC 交易。

在以下情況下，鏈必須拒絕交易並且不進行狀態轉換：

- IBC 交易無法反序列化。
- IBC 交易期望的是對方鏈創建跨鏈帳戶以外的簽名者。

不限制你如何區分一個簽名者是不是對方鏈的。但是最常見的方法是在註冊鏈間帳戶時記錄帳戶在狀態中，並驗證簽名者是否是記錄的鏈間帳戶。

### 數據結構

每個鏈必須實現以下介面以支持鏈間帳戶。 `IBCAccountModule`介面的`createOutgoingPacket`方法定義了創建特定類型的傳出封包的方式。類型指示應如何為主機鏈構建和序列化 IBC 帳戶交易。通常，類型指示主機鏈的構建框架。 `generateAddress`定義了如何使用標識符和鹽（salt）確定帳戶地址的方法。建議使用鹽生成地址，但不是必需的。如果該鏈不支持用確定性的方式來生成帶有鹽的地址，則可以以其自己的方式來生成。 `createAccount`使用生成的地址創建帳戶。新的鏈間帳戶不得與現有帳戶衝突，並且鏈應跟蹤是哪個對方鏈創建的新的鏈間帳戶，以驗證`authenticateTx`中交易簽名人的權限。 `authenticateTx`驗證交易並檢查交易中的簽名者是否具有正確的權限。成功通過身份認證後， `runTx`執行交易。

```typescript
type Tx = object

interface IBCAccountModule {
  createOutgoingPacket(chainType: Uint8Array, data: any)
  createAccount(address: Uint8Array)
  generateAddress(identifier: Identifier, salt: Uint8Array): Uint8Array
  deserialiseTx(txBytes: Uint8Array): Tx
  authenticateTx(tx: Tx): boolean
  runTx(tx: Tx): uint32
}
```

對方鏈使用`RegisterIBCAccountPacketData`註冊帳戶。使用通道標識符和鹽可以確定性的定義鏈間帳戶的地址。 `generateAccount`方法用於生成新的鏈間帳戶的地址。建議通過`hash(identifier+salt)`生成地址，但是也可以使用其他方法。此函數必須通過標識符和鹽生成唯一的確定性地址。

```typescript
interface RegisterIBCAccountPacketData {
  salt: Uint8Array
}
```

`RunTxPacketData`用於在鏈間帳戶上執行交易。交易位元組包含交易本身，並以適合於目標鏈的方式進行序列化。

```typescript
interface RunTxPacketData {
  txBytes: Uint8Array
}
```

`IBCAccountHandler`介面允許源連結收在鏈間帳戶上執行交易的結果。

```typescript
interface InterchainTxHandler {
  onAccountCreated(identifier: Identifier, address: Address)
  onTxSucceeded(identifier: Identifier, txBytes: Uint8Array)
  onTxFailed(identifier: Identifier, txBytes: Uint8Array, errorCode: Uint8Array)
}
```

### 子協議

本文所述的子協議應在“鏈間帳戶橋”模組中實現，並可以訪問應用的路由和編解碼器（解碼器或解組器），和訪問 IBC 路由模組。

### 埠和通道設置

創建模組時（可能是在初始化區塊鏈本身時），必須調用一次`setup`函數，以綁定到適當的埠並創建託管地址（為模組擁有）。

```typescript
function setup() {
  relayerModule.bindPort("interchain-account", ModuleCallbacks{
    onChanOpenInit,
    onChanOpenTry,
    onChanOpenAck,
    onChanOpenConfirm,
    onChanCloseInit,
    onChanCloseConfirm,
    onSendPacket,
    onRecvPacket,
    onTimeoutPacket,
    onAcknowledgePacket,
    onTimeoutPacketClose
  })
}
```

調用`setup`功能後，即可通過 IBC 路由模組在獨立鏈上的鏈間帳戶模組實例之間創建通道。

管理員（具有在主機狀態機上創建連接和通道的權限）負責建立與其他狀態機的連接，並創建與其他鏈上該模組（或支持此介面的另一個模組）的其他實例的通道。該規範僅定義了封包處理語義，並以這樣一種方式定義它們：模組本身無需擔心在任何時間點可能存在或不存在哪些連接或通道。

### 路由模組回調

### 通道生命週期管理

當且僅當以下情況時，機器`A`和`B`接受另一台機器上任何模組的新通道：

- 另一個模組綁定到“鏈間帳戶”埠。
- 正在創建的通道是有序的。
- 版本字串為空。

```typescript
function onChanOpenInit(
  order: ChannelOrder,
  connectionHops: [Identifier],
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyPortIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  version: string) {
  // only ordered channels allowed
  abortTransactionUnless(order === ORDERED)
  // only allow channels to "interchain-account" port on counterparty chain
  abortTransactionUnless(counterpartyPortIdentifier === "interchain-account")
  // version not used at present
  abortTransactionUnless(version === "")
}
```

```typescript
function onChanOpenTry(
  order: ChannelOrder,
  connectionHops: [Identifier],
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyPortIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  version: string,
  counterpartyVersion: string) {
  // only ordered channels allowed
  abortTransactionUnless(order === ORDERED)
  // version not used at present
  abortTransactionUnless(version === "")
  abortTransactionUnless(counterpartyVersion === "")
  // only allow channels to "interchain-account" port on counterparty chain
  abortTransactionUnless(counterpartyPortIdentifier === "interchain-account")
}
```

```typescript
function onChanOpenAck(
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  version: string) {
  // version not used at present
  abortTransactionUnless(version === "")
  // port has already been validated
}
```

```typescript
function onChanOpenConfirm(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
  // accept channel confirmations, port has already been validated
}
```

```typescript
function onChanCloseInit(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
  // no action necessary
}
```

```typescript
function onChanCloseConfirm(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
  // no action necessary
}
```

### 封包中繼

用簡單的文字描述就是`A`和`B`之間，鏈 A 想要在鏈 B 上註冊一個鏈間帳戶並控制它。同樣，也可以反過來。

```typescript
function onRecvPacket(packet: Packet): bytes {
  if (packet.data is RunTxPacketData) {
    const tx = deserialiseTx(packet.data.txBytes)
    abortTransactionUnless(authenticateTx(tx))
    return runTx(tx)
  }

  if (packet.data is RegisterIBCAccountPacketData) {
    RegisterIBCAccountPacketData data = packet.data
    identifier = "{packet/sourcePort}/{packet.sourceChannel}"
    const address = generateAddress(identifier, packet.salt)
    createAccount(address)
    // Return generated address.
    return address
  }

  return 0x
}
```

```typescript
function onAcknowledgePacket(
  packet: Packet,
  acknowledgement: bytes) {
  if (packet.data is RegisterIBCAccountPacketData)
    if (acknowledgement !== 0x) {
      identifier = "{packet/sourcePort}/{packet.sourceChannel}"
      onAccountCreated(identifier, acknowledgement)
    }
  if (packet.data is RunTxPacketData) {
    identifier = "{packet/destPort}/{packet.destChannel}"
    if (acknowledgement === 0x)
        onTxSucceeded(identifier: Identifier, packet.data.txBytes)
    else
        onTxFailed(identifier: Identifier, packet.data.txBytes, acknowledgement)
  }
}
```

```typescript
function onTimeoutPacket(packet: Packet) {
  // Receiving chain should handle this event as if the tx in packet has failed
  if (packet.data is RunTxPacketData) {
    identifier = "{packet/destPort}/{packet.destChannel}"
    // 0x99 error code means timeout.
    onTxFailed(identifier: Identifier, packet.data.txBytes, 0x99)
  }
}
```

```typescript
function onTimeoutPacketClose(packet: Packet) {
  // nothing is necessary
}
```

## 向後相容性

不適用。

## 向前相容性

不適用。

## 範例實現

cosmos-sdk 的虛擬碼：https://github.com/everett-protocol/everett-hackathon/tree/master/x/interchain-account 以太坊上的鏈間帳戶的 POC：https://github.com/everett-protocol/ethereum-interchain-account

## 其他實現

（其他實現的連結或描述）

## 歷史

2019年8月1日-討論了概念

2019年9月24日-建議草案

2019年11月8日-重大修訂

2019年12月2日-較小修訂（在以太坊上添加更多具體描述並添加鏈間帳戶）

## 版權

本文中的所有內容均根據 [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0) 獲得許可。
