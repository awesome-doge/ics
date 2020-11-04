---
ics: 20
title: 同質通證轉移
stage: 草案
category: IBC/APP
requires: 25, 26
kind: 實例化
author: Christopher Goes <cwgoes@interchain.berlin>
created: 2019-07-15
modified: 2020-02-24
---

## 概覽

該標準規定了通過 IBC 通道在各自鏈上的兩個模組之間進行通證轉移的封包的數據結構，狀態機處理邏輯以及編碼細節。本文所描述的狀態機邏輯允許在無許可通道打開的情況下安全的處理多個鏈的通證。該邏輯通過在節點狀態機上的 IBC 路由模組和一個現存的資產跟蹤模組之間建立實現了一個同質通證轉移的橋接模組。

### 動機

基於 IBC 協議連接的一組鏈的用戶可能希望在一條鏈上能利用在另一條鏈上發行的資產來使用該鏈上的附加功能，例如交易或隱私保護，同時保持發行鏈上的原始資產的同質性。該應用層標準描述了一個在基於 IBC 連接的鏈間轉移同質通證的協議，該協議保留了資產的同質性和資產所有權，限制了拜占庭錯誤的影響，並且無需額外許可。

### 定義

[ICS 25](../ics-025-handler-interface) 和 [ICS 26](../ics-026-routing-module) 分別定義了 IBC 處理介面和 IBC 路由模組介面。

### 所需屬性

- 保持同質性（雙向錨定）。
- 保持供應量不變（在單一源鏈和模組上保持不變或通脹）。
- 無許可的通證轉移，無需將連接（connections）、模組或通證面額加入白名單。
- 對稱（所有鏈實現相同的邏輯，hubs 和 zones 無協議差別）。
- 容錯：防止由於鏈`B`的拜占庭行為造成源自鏈`A`的通證的拜占庭通貨膨脹（儘管任何將通證轉移到鏈`B`上的用戶都面臨風險）。

## 技術規範

### 數據結構

僅需要一個封包數據類型`FungibleTokenPacketData`，該類型指定了面額，數量，發送帳戶，接受帳戶以及發送鏈是否為資產的發行鏈。

```typescript
interface FungibleTokenPacketData {
  denomination: string
  amount: uint256
  sender: string
  receiver: string
}
```

確認數據類型描述轉帳是成功還是失敗，以及失敗的原因（如果有）。

```typescript
interface FungibleTokenPacketAcknowledgement {
  success: boolean
  error: Maybe<string>
}
```

同質通證轉移橋接模組跟蹤與狀態中指定通道關聯的託管地址。假設`ModuleState`的欄位在範圍內。

```typescript
interface ModuleState {
  channelEscrowAddresses: Map<Identifier, string>
}
```

### 子協議

本文所述的子協議應該在“同質通證轉移橋接”模組中實現，並且可以訪問 band 模組和 IBC 路由模組。

#### 埠 & 通道設置

當創建“同質通證轉移橋接”模組時（也可能是區塊鏈本身初始化時），必須僅調用一次`setup`函數用於綁定到對應的埠並創建一個託管地址（該地址由模組所有）。

```typescript
function setup() {
  capability = routingModule.bindPort("bank", ModuleCallbacks{
    onChanOpenInit,
    onChanOpenTry,
    onChanOpenAck,
    onChanOpenConfirm,
    onChanCloseInit,
    onChanCloseConfirm,
    onRecvPacket,
    onTimeoutPacket,
    onAcknowledgePacket,
    onTimeoutPacketClose
  })
  claimCapability("port", capability)
}
```

調用`setup`函數後，通過在不同鏈上的同質通證轉移模組之間的 IBC 路由模組創建通道。

管理員（具有在節點的狀態機上創建連接和通道的權限）負責在本地鏈與其他鏈的狀態機之間創建連接，在本地鏈與其他鏈的該模組（或支持該介面的其他模組）的實例之間創建通道。本規範僅定義了封包處理語義，模組本身在任意時間點都無需關心連接或通道是否存在。

#### 路由模組回調

##### 通道生命週期管理

機器`A`和機器`B`在當且僅當以下情況下接受來自第三台機器上任何模組的新通道創建請求：

- 第三台機器的模組綁定到“bank”埠。
- 創建的通道是無序的。
- 版本號為空。

```typescript
function onChanOpenInit(
  order: ChannelOrder,
  connectionHops: [Identifier],
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyPortIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  version: string) {
  // only unordered channels allowed
  abortTransactionUnless(order === UNORDERED)
  // only allow channels to "bank" port on counterparty chain
  abortTransactionUnless(counterpartyPortIdentifier === "bank")
  // assert that version is "ics20-1"
  abortTransactionUnless(version === "ics20-1")
  // allocate an escrow address
  channelEscrowAddresses[channelIdentifier] = newAddress()
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
  // only unordered channels allowed
  abortTransactionUnless(order === UNORDERED)
  // assert that version is "ics20-1"
  abortTransactionUnless(version === "ics20-1")
  abortTransactionUnless(counterpartyVersion === "ics20-1")
  // only allow channels to "bank" port on counterparty chain
  abortTransactionUnless(counterpartyPortIdentifier === "bank")
  // allocate an escrow address
  channelEscrowAddresses[channelIdentifier] = newAddress()
}
```

```typescript
function onChanOpenAck(
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  version: string) {
  // port has already been validated
  // assert that version is "ics20-1"
  abortTransactionUnless(version === "ics20-1")
}
```

```typescript
function onChanOpenConfirm(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
  // accept channel confirmations, port has already been validated, version has already been validated
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

##### 封包中繼

用簡單文字來描述就是，在 `A` 和`B`兩個鏈間：

- 在源 zone 上，橋接模組會在發送鏈上託管現有的本地資產面額，並在接收鏈上生成憑證。
- 在目標 zone 上，橋接模組會在發送鏈上銷毀本地憑證，並在接收鏈上解除對本地資產面額的託管。
- 當封包超時時，本地資產將解除託管並退還給發送者，或將憑證發回給發送者。
- 確認數據用於處理失敗，例如無效面額或無效目標帳戶。返回失敗的確認比終止交易更可取，因為它更容易使發送鏈根據失敗的本質而採取適當的措施。

模組中對節點狀態機上的帳戶所有者進行簽名檢查的交易處理程序必須調用`createOutgoingPacket`。

```typescript
function createOutgoingPacket(
  denomination: string,
  amount: uint256,
  sender: string,
  receiver: string,
  destPort: string,
  destChannel: string,
  sourcePort: string,
  sourceChannel: string,
  timeoutHeight: uint64,
  timeoutTimestamp: uint64) {
  // inspect the denomination to determine whether or not we are the source chain
  prefix = "{destPort}/{destChannel}"
  source = denomination.slice(0, len(prefix)) === prefix
  if source {
    // sender is source chain: escrow tokens
    // determine escrow account
    escrowAccount = channelEscrowAddresses[packet.sourceChannel]
    // escrow source tokens (assumed to fail if balance insufficient)
    bank.TransferCoins(sender, escrowAccount, denomination.slice(len(prefix)), amount)
  } else {
    // receiver is source chain, burn vouchers
    // construct receiving denomination, check correctness
    prefix = "{sourcePort}/{sourceChannel}"
    abortTransactionUnless(denomination.slice(0, len(prefix)) === prefix)
    // burn vouchers (assumed to fail if balance insufficient)
    bank.BurnCoins(sender, denomination, amount)
  }
  FungibleTokenPacketData data = FungibleTokenPacketData{denomination, amount, sender, receiver}
  handler.sendPacket(Packet{timeoutHeight, timeoutTimestamp, destPort, destChannel, sourcePort, sourceChannel, data}, getCapability("port"))
}
```

當路由模組收到一個封包後調用`onRecvPacket`。

```typescript
function onRecvPacket(packet: Packet) {
  FungibleTokenPacketData data = packet.data
  // inspect the denomination to determine whether or not we are the source chain
  prefix = "{packet.destPort}/{packet.destChannel}"
  source = denomination.slice(0, len(prefix)) === prefix
  // construct default acknowledgement of success
  FungibleTokenPacketAcknowledgement ack = FungibleTokenPacketAcknowledgement{true, null}
  if source {
    // sender was source, mint vouchers to receiver (assumed to fail if balance insufficient)
    err = bank.MintCoins(data.receiver, data.denomination, data.amount)
    if (err !== nil)
      ack = FungibleTokenPacketAcknowledgement{false, "mint coins failed"}
  } else {
    // receiver is source chain: unescrow tokens
    // determine escrow account
    escrowAccount = channelEscrowAddresses[packet.destChannel]
    // construct receiving denomination, check correctness
    prefix = "{packet/sourcePort}/{packet.sourceChannel}"
    if (data.denomination.slice(0, len(prefix)) !== prefix)
      ack = FungibleTokenPacketAcknowledgement{false, "invalid denomination"}
    else {
      // unescrow tokens to receiver (assumed to fail if balance insufficient)
      err = bank.TransferCoins(escrowAccount, data.receiver, data.denomination.slice(len(prefix)), data.amount)
      if (err !== nil)
        ack = FungibleTokenPacketAcknowledgement{false, "transfer coins failed"}
    }
  }
  return ack
}
```

當由路由模組發送的封包被確認後，該模組調用`onAcknowledgePacket`。

```typescript
function onAcknowledgePacket(
  packet: Packet,
  acknowledgement: bytes) {
  // if the transfer failed, refund the tokens
  if (!ack.success)
    refundTokens(packet)
}
```

當由路由模組發送的封包超時（例如封包沒有被目標連結收到）後，路由模組調用`onTimeoutPacket`。

```typescript
function onTimeoutPacket(packet: Packet) {
  // the packet timed-out, so refund the tokens
  refundTokens(packet)
}
```

`refundTokens`會在兩處被調用，失敗時的`onAcknowledgePacket` 和`onTimeoutPacket`，用來退還託管的通證給原始發送者。

```typescript
function refundTokens(packet: Packet) {
  FungibleTokenPacketData data = packet.data
  prefix = "{packet.destPort}/{packet.destChannel}"
  source = data.denomination.slice(0, len(prefix)) === prefix
  if source {
    // sender was source chain, unescrow tokens
    // determine escrow account
    escrowAccount = channelEscrowAddresses[packet.destChannel]
    // construct receiving denomination, check correctness
    // unescrow tokens back to sender
    bank.TransferCoins(escrowAccount, data.sender, data.denomination.slice(len(prefix)), data.amount)
  } else {
    // receiver was source chain, mint vouchers
    // construct receiving denomination, check correctness
    prefix = "{packet.sourcePort}/{packet.sourceChannel}"
    // we abort here because we couldn't have sent this packet
    abortTransactionUnless(data.denomination.slice(0, len(prefix)) === prefix)
    // mint vouchers back to sender
    bank.MintCoins(data.sender, data.denomination, data.amount)
  }
}
```

```typescript
function onTimeoutPacketClose(packet: Packet) {
  // can't happen, only unordered channels allowed
}
```

#### 原理

##### 正確性

該實現保持了同質性和供應量不變。

同質性：如果通證已發送到目標鏈，則可以以相同面額和數量兌換回源鏈。

供應量：將供應重新定義為未鎖定的通證。所有源鏈的發送量等於目標鏈的接受量。源鏈可以改變通證的供應量。

##### 多鏈注意事項

此規範不能直接處理“菱形問題”，在該問題中，用戶將源自鏈 A 的通證發送到鏈 B，然後又發送給鏈 D，並希望通過 D-> C-> A 歸還它，由於此時通證的供應量被認為是由鏈 B 控制（面額將為“ {portOnD} / {channelOnD} / {portOnB} / {channelOnB} / denom”），鏈 C 不能充當中介。尚不清楚該場景是否應按協議處理—可能只需要原路返回就可以了（如果在這兩個途徑上都有頻繁的流動性和一定的結餘，菱形路徑將在大多數情況下適用）。較長的贖回路徑引起的複雜性可能導致網路拓撲結構中出現中心鏈。

為了跟蹤沿著各種路徑在鏈網路中移動的所有面額，對於特定的鏈實現一個註冊表將有助於跟蹤每個面額的“全局”源鏈。最終用戶服務提供商（例如錢包作者）可能希望集成這樣的註冊表，或保留自己的典範源鏈和人類可讀名稱的映射，以改善 UX。

#### 可選附錄

- 每個本地鏈都可以選擇保留一個查找表，以在狀態中使用簡短，用戶友好的本地面額，在發送和接收封包時，它們會與較長面額進行轉換。
- 可能會對與哪些其他機器連接以及建立哪些通道施加其他限制。

## 向後相容性

不適用。

## 向前相容性

此初始標準在通道握手中使用版本“ ics20-1”。

該標準的未來版本可以在通道握手中使用其他版本，並安全的更改封包數據格式和封包處理程序的語義。

## 範例實現

即將到來。

## 其他實現

即將到來。

## 歷史

2019年7月15 - 草案完成

2019年7月29 - 主要修訂；整理

2019年8月25 - 主要修訂；進一步整理

2020年2月3日-進行修訂，以處理對成功和失敗的確認

2020年2月24日-用來推斷來源欄位的修訂，包括版本字串

## 版權

本文中的所有內容均根據 [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0) 獲得許可。
