---
ics: 4
title: 通道和封包語義
stage: 草案
category: IBC/TAO
kind: 實例化
requires: 2, 3, 5, 24
author: Christopher Goes <cwgoes@tendermint.com>
created: 2019-03-07
modified: 2019-08-25
---

## 概要

“通道”抽象為區塊鏈間通信協議提供消息傳遞語義，分為三類：排序、僅一次傳遞和模組許可。通道充當封包在一條鏈上的模組與另一條鏈上的模組之間傳遞的通道，從而確保封包僅被執行一次，按照其發送順序進行傳遞（如有必要），並僅傳遞給擁有目標鏈上通道的另一端的相應的模組。每個通道都與一個特定的連接關聯，並且一個連接可以具有任意數量的關聯通道，從而允許使用公共標識符，並利用連接和輕用戶端在所有通道上分攤區塊頭驗證的成本。

通道不關心其中傳遞的內容。發送和接收 IBC 封包的模組決定如何構造封包數據，以及如何對傳入的封包數據進行操作，並且必須利用其自身的應用程式邏輯來根據封包中的數據來確定要應用的狀態轉換。

### 動機

區塊鏈間通信協議使用跨鏈消息傳遞模型。 外部中繼器進程將 IBC *封包*從一條鏈中繼到另一條鏈。鏈`A`和鏈`B`獨立的確認新的塊，並且從一個鏈到另一個鏈的封包可能會被任意延遲、審查或重新排序。封包對於中繼器是可見的，並且可以被任何中繼器進程讀取，然後被提交給任何其他鏈。

> IBC 協議必須保證順序（對於有序通道）和僅有一次傳遞，以允許應用程式探討兩條鏈上已連接模組的組合狀態。例如，一個應用程式可能希望允許單個通證化的資產在多個區塊鏈之間轉移並保留在多個區塊鏈上，同時保留同質化和供應量。當特定的 IBC 封包提交到鏈`B`時，應用程式可以在鏈`B`上鑄造資產憑據，並要求鏈`A`將等額的資產託管在鏈`A`上，直到以後以相反的 IBC 封包將憑證兌換回鏈`A`為止。這種順序保證配合正確的應用邏輯，可以確保兩個鏈上的資產總量不變，並且在鏈`B`上鑄造的任何資產憑證都可以之後兌換回鏈`A`上。

為了向應用層提供所需的排序、僅有一次傳遞和模組許可語義，區塊鏈間通信協議必須實現一種抽象以強制執行這些語義——通道就是這種抽象。

### 定義

`ConsensusState` 在 [ICS 2](../ics-002-client-semantics) 中被定義.

`Connection` 在 [ICS 3](../ics-003-connection-semantics) 中被定義.

`Port`和`authenticateCapability`在 [ICS 5](../ics-005-port-allocation) 中被定義。

`hash`是一種通用的抗碰撞哈希函數，其細節必須由使用通道的模組商定。 `hash`在不同的鏈可以有不同的定義。

`Identifier` ， `get` ， `set` ， `delete` ， `getCurrentHeight`和模組系統相關的原語在 [ICS 24](../ics-024-host-requirements) 中被定義。

*通道*是用於在單獨的區塊鏈上的特定模組之間進行僅一次封包傳遞的管道，該模組至少具備封包發送端和封包接收端。

*雙向*通道是封包可以在兩個方向上流動的通道：從`A`到`B`和從`B`到`A`

*單向*通道是指封包只能沿一個方向流動的通道：從`A`到`B` （或從`B`到`A` ，命名的順序是任意的）。

*有序*通道是指完全按照發送順序傳遞封包的通道。

*無序*通道是指可以以任何順序傳遞封包的通道，該順序可能與封包的發送順序不同。

```typescript
enum ChannelOrder {
  ORDERED,
  UNORDERED,
}
```

方向和順序是無關的，因此可以說雙向無序通道，單向有序通道等。

所有通道均提供有且僅有一次的封包傳遞，這意味著在通道的一端發送的封包最終將不多於且不少於一次的傳遞到另一端。

該規範僅涉及*雙向*通道。*單向*通道可以使用幾乎完全相同的協議，並將在以後的 ICS 中進行概述。

通道端是一條鏈上存儲通道元數據的數據結構：

```typescript
interface ChannelEnd {
  state: ChannelState
  ordering: ChannelOrder
  counterpartyPortIdentifier: Identifier
  counterpartyChannelIdentifier: Identifier
  connectionHops: [Identifier]
  version: string
}
```

- `state`是通道端的當前狀態。
- `ordering`欄位指示通道是有序的還是無序的。
- `counterpartyPortIdentifier`標識通道另一端的對應鏈上的埠號。
- `counterpartyChannelIdentifier`標識對應鏈的通道端。
- `nextSequenceSend`是單獨存儲的，追蹤下一個要發送的封包的序號。
- `nextSequenceRecv`是單獨存儲的，追蹤要接收的下一個封包的序號。
- `nextSequenceAck`是單獨存儲的，追蹤要確認的下一個封包的序號。
- `connectionHops`按順序的存儲在此通道上發送的封包將途徑的連接標識符列表。目前，此列表的長度必須為 1。將來可能會支持多跳通道。
- `version`字串存儲一個不透明的通道版本號，該版本號在握手期間已達成共識。這可以確定模組級別的配置，例如通道使用哪種封包編碼。核心 IBC 協議不會使用該版本號。

通道端具有以下*狀態* ：

```typescript
enum ChannelState {
  INIT,
  TRYOPEN,
  OPEN,
  CLOSED,
}
```

- 處於`INIT`狀態的通道端，表示剛剛開始了握手的建立。
- 處於`TRYOPEN`狀態的通道端表示已確認對方鏈的握手。
- 處於`OPEN`狀態的通道端，表示已完成握手，並為發送和接收封包作好了準備。
- 處於`CLOSED`狀態的通道端，表示通道已關閉，不能再用於發送或接收封包。

區塊鏈間通信協議中的`Packet`是如下定義的特定介面：

```typescript
interface Packet {
  sequence: uint64
  timeoutHeight: uint64
  timeoutTimestamp: uint64
  sourcePort: Identifier
  sourceChannel: Identifier
  destPort: Identifier
  destChannel: Identifier
  data: bytes
}
```

- `sequence`對應於發送和接收的順序，其中序號靠前的封包必須比序號靠後的封包先發送或接收。
- `timeoutHeight`指示目標鏈上的一個共識高度，此高度後不再處理封包，而是計為已超時。
- `timeoutTimestamp`指示目標鏈上的一個時間戳，此後將不再處理封包，而是計為已超時。
- `sourcePort`標識發送鏈上的埠。
- `sourceChannel`標識發送鏈上的通道端。
- `destPort`標識接收鏈上的埠。
- `destChannel`標識接收鏈上的通道端。
- `data`是不透明的值，可以由關聯模組的應用程式邏輯定義。

請注意，`Packet`絕不會直接序列化。而是在某些函數調用中使用的中間結構，可能需要由調用 IBC 處理程序的模組來創建或處理該中間結構。

`OpaquePacket`是一個封包，但是被主機狀態機掩蓋為一種模糊的數據類型，因此，除了將其傳遞給 IBC 處理程序之外，模組無法對其進行任何操作。IBC 處理程序可以將`Packet`轉換為`OpaquePacket` ，或反過來。

```typescript
type OpaquePacket = object
```

### 所需屬性

#### 效率

- 封包傳輸和確認的速度應僅受底層鏈速度的限制。證明應儘可能是批次化的。

#### 僅一次傳遞

- 在通道的一端發送的 IBC 封包應僅一次的傳遞到另一端。
- 對於僅一次的安全性，不需要網路同步假設。如果其中一條鏈或兩條鏈都掛起了，則封包最多傳遞不超過一次，並且一旦鏈恢復，封包就應該能夠再次流轉。

#### 排序

- 在有序通道上，應按相同的順序發送和接收封包：如果封包 *x* 在鏈`A`通道端的封包 *y* 之前發送，則封包 *x* 必須在相應的鏈`B`通道端的封包 *y* 之前收到。
- 在無序通道上，可以以任何順序發送和接收封包。像有序封包一樣，無序封包具有單獨的根據目標鏈高度指定的超時高度。

#### 許可

- 通道應該在握手期間被通道兩端的模組許可，並且此後不可變更（更高級別的邏輯可以透過標記埠的所有權來標記通道所有權）。只有與通道端關聯的模組才能在其上發送或接收封包。

## 技術指標

### 數據流可視化

用戶端、連接、通道和封包的體系結構：

![Dataflow Visualisation](../../../../spec/ics-004-channel-and-packet-semantics/dataflow.png)

### 預備知識

#### 存儲路徑

通道的結構存儲在一個結合了埠標識符和通道標識符的唯一存儲路徑前綴下：

```typescript
function channelPath(portIdentifier: Identifier, channelIdentifier: Identifier): Path {
    return "ports/{portIdentifier}/channels/{channelIdentifier}"
}
```

與通道關聯的能力鍵存儲在`channelCapabilityPath` ：

```typescript
function channelCapabilityPath(portIdentifier: Identifier, channelIdentifier: Identifier): Path {
  return "{channelPath(portIdentifier, channelIdentifier)}/key"
}
```

無符號整數計數器`nextSequenceSend` ， `nextSequenceRecv`和`nextSequenceAck`是分別存儲的，因此可以單獨證明它們：

```typescript
function nextSequenceSendPath(portIdentifier: Identifier, channelIdentifier: Identifier): Path {
    return "{channelPath(portIdentifier, channelIdentifier)}/nextSequenceSend"
}

function nextSequenceRecvPath(portIdentifier: Identifier, channelIdentifier: Identifier): Path {
    return "{channelPath(portIdentifier, channelIdentifier)}/nextSequenceRecv"
}

function nextSequenceAckPath(portIdentifier: Identifier, channelIdentifier: Identifier): Path {
    return "{channelPath(portIdentifier, channelIdentifier)}/nextSequenceAck"
}
```

固定大小的加密承諾封包數據欄位存儲在封包序號下：

```typescript
function packetCommitmentPath(portIdentifier: Identifier, channelIdentifier: Identifier, sequence: uint64): Path {
    return "{channelPath(portIdentifier, channelIdentifier)}/packets/" + sequence
}
```

存儲中缺失的路徑相當於占用零位。

封包確認數據存儲在`packetAcknowledgementPath` ：

```typescript
function packetAcknowledgementPath(portIdentifier: Identifier, channelIdentifier: Identifier, sequence: uint64): Path {
    return "{channelPath(portIdentifier, channelIdentifier)}/acknowledgements/" + sequence
}
```

無序通道必須始終向該路徑寫入確認訊息（甚至是空的），使得此類確認訊息的缺失可以用作超時證明。有序通道也可以寫一個確認訊息，但不是必須的。

### 版本控制

在握手過程中，通道的兩端在與該通道關聯的版本位元組串上達成一致。 此版本位元組串的內容對於 IBC 核心協議保持不透明。 狀態機主機可以利用版本數據來標示其支持的 IBC/APP 協議，確認封包編碼格式，或在協商與 IBC 協議之上自訂邏輯有關的其他通道元數據。

狀態機主機可以安全的忽略版本數據或指定一個空字串。

### 子協議

> 注意：如果主機狀態機正在使用對象能力認證（請參閱 [ICS 005](../ics-005-port-allocation) ），則所有使用埠的功能都將帶有附加能力參數。

#### 標識符驗證

通道存儲在唯一的`(portIdentifier, channelIdentifier)`前綴下。 可以提供驗證函數`validatePortIdentifier` 。

```typescript
type validateChannelIdentifier = (portIdentifier: Identifier, channelIdentifier: Identifier) => boolean
```

如果未提供，預設的`validateChannelIdentifier`函數將始終返回`true` 。

#### 通道生命週期管理

![Channel State Machine](channel-state-machine.png)

發起人 | 數據報 | 作用鏈 | 之前狀態 (A, B) | 之後狀態 (A, B)
--- | --- | --- | --- | ---
參與者 | ChanOpenInit | A | (none, none) | (INIT, none)
中繼器 | ChanOpenTry | B | (INIT, none) | (INIT, TRYOPEN)
中繼器 | ChanOpenAck | A | (INIT, TRYOPEN) | (OPEN, TRYOPEN)
中繼器 | ChanOpenConfirm | B | (OPEN, TRYOPEN) | (OPEN, OPEN)

發起人 | 數據報 | 作用鏈 | 之前狀態 (A, B) | 之後狀態 (A, B)
--- | --- | --- | --- | ---
參與者 | ChanCloseInit | A | (OPEN, OPEN) | (CLOSED, OPEN)
中繼器 | ChanCloseConfirm | B | (CLOSED, OPEN) | (CLOSED, CLOSED)

##### 建立握手

與另一個鏈上的模組發起通道建立握手的模組調用`chanOpenInit`函數。

建立通道必須提供本地通道標識符、本地埠、遠程埠和遠程通道的標識符。

當建立握手完成後，發起握手的模組將擁有在帳本上已創建通道的一端，而對應的另一條鏈的模組將擁有通道的另一端。創建通道後，所有權就無法更改（儘管更高級別的抽象可以實現並提供此功能）。

```typescript
function chanOpenInit(
  order: ChannelOrder,
  connectionHops: [Identifier],
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyPortIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  version: string): CapabilityKey {
    abortTransactionUnless(validateChannelIdentifier(portIdentifier, channelIdentifier))

    abortTransactionUnless(connectionHops.length === 1) // for v1 of the IBC protocol

    abortTransactionUnless(provableStore.get(channelPath(portIdentifier, channelIdentifier)) === null)
    connection = provableStore.get(connectionPath(connectionHops[0]))

    // optimistic channel handshakes are allowed
    abortTransactionUnless(connection !== null)
    abortTransactionUnless(authenticateCapability(portPath(portIdentifier), portCapability))
    channel = ChannelEnd{INIT, order, counterpartyPortIdentifier,
                         counterpartyChannelIdentifier, connectionHops, version}
    provableStore.set(channelPath(portIdentifier, channelIdentifier), channel)
    channelCapability = newCapability(channelCapabilityPath(portIdentifier, channelIdentifier))
    provableStore.set(nextSequenceSendPath(portIdentifier, channelIdentifier), 1)
    provableStore.set(nextSequenceRecvPath(portIdentifier, channelIdentifier), 1)
    provableStore.set(nextSequenceAckPath(portIdentifier, channelIdentifier), 1)
    return channelCapability
}
```

模組調用`chanOpenTry`函數，以接受由另一條鏈上的模組發起的通道建立握手的第一步。

```typescript
function chanOpenTry(
  order: ChannelOrder,
  connectionHops: [Identifier],
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyPortIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  version: string,
  counterpartyVersion: string,
  proofInit: CommitmentProof,
  proofHeight: uint64): CapabilityKey {
    abortTransactionUnless(validateChannelIdentifier(portIdentifier, channelIdentifier))
    abortTransactionUnless(connectionHops.length === 1) // for v1 of the IBC protocol
    previous = provableStore.get(channelPath(portIdentifier, channelIdentifier))
    abortTransactionUnless(
      (previous === null) ||
      (previous.state === INIT &&
       previous.order === order &&
       previous.counterpartyPortIdentifier === counterpartyPortIdentifier &&
       previous.counterpartyChannelIdentifier === counterpartyChannelIdentifier &&
       previous.connectionHops === connectionHops &&
       previous.version === version)
      )
    abortTransactionUnless(authenticateCapability(portPath(portIdentifier), portCapability))
    connection = provableStore.get(connectionPath(connectionHops[0]))
    abortTransactionUnless(connection !== null)
    abortTransactionUnless(connection.state === OPEN)
    expected = ChannelEnd{INIT, order, portIdentifier,
                          channelIdentifier, [connection.counterpartyConnectionIdentifier], counterpartyVersion}
    abortTransactionUnless(connection.verifyChannelState(
      proofHeight,
      proofInit,
      counterpartyPortIdentifier,
      counterpartyChannelIdentifier,
      expected
    ))
    channel = ChannelEnd{TRYOPEN, order, counterpartyPortIdentifier,
                         counterpartyChannelIdentifier, connectionHops, version}
    provableStore.set(channelPath(portIdentifier, channelIdentifier), channel)
    channelCapability = newCapability(channelCapabilityPath(portIdentifier, channelIdentifier))
    provableStore.set(nextSequenceSendPath(portIdentifier, channelIdentifier), 1)
    provableStore.set(nextSequenceRecvPath(portIdentifier, channelIdentifier), 1)
    provableStore.set(nextSequenceAckPath(portIdentifier, channelIdentifier), 1)
    return channelCapability
}
```

握手發起模組調用`chanOpenAck` ，以確認對方鏈的模組已接受發起的請求。

```typescript
function chanOpenAck(
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyVersion: string,
  proofTry: CommitmentProof,
  proofHeight: uint64) {
    channel = provableStore.get(channelPath(portIdentifier, channelIdentifier))
    abortTransactionUnless(channel.state === INIT || channel.state === TRYOPEN)
    abortTransactionUnless(authenticateCapability(channelCapabilityPath(portIdentifier, channelIdentifier), capability))
    connection = provableStore.get(connectionPath(channel.connectionHops[0]))
    abortTransactionUnless(connection !== null)
    abortTransactionUnless(connection.state === OPEN)
    expected = ChannelEnd{TRYOPEN, channel.order, portIdentifier,
                          channelIdentifier, [connection.counterpartyConnectionIdentifier], counterpartyVersion}
    abortTransactionUnless(connection.verifyChannelState(
      proofHeight,
      proofTry,
      channel.counterpartyPortIdentifier,
      channel.counterpartyChannelIdentifier,
      expected
    ))
    channel.state = OPEN
    channel.version = counterpartyVersion
    provableStore.set(channelPath(portIdentifier, channelIdentifier), channel)
}
```

握手接受模組調用`chanOpenConfirm`函數以確認在另一條鏈上進行握手發起模組的確認訊息，並完成通道創建握手。

```typescript
function chanOpenConfirm(
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  proofAck: CommitmentProof,
  proofHeight: uint64) {
    channel = provableStore.get(channelPath(portIdentifier, channelIdentifier))
    abortTransactionUnless(channel !== null)
    abortTransactionUnless(channel.state === TRYOPEN)
    abortTransactionUnless(authenticateCapability(channelCapabilityPath(portIdentifier, channelIdentifier), capability))
    connection = provableStore.get(connectionPath(channel.connectionHops[0]))
    abortTransactionUnless(connection !== null)
    abortTransactionUnless(connection.state === OPEN)
    expected = ChannelEnd{OPEN, channel.order, portIdentifier,
                          channelIdentifier, [connection.counterpartyConnectionIdentifier], channel.version}
    abortTransactionUnless(connection.verifyChannelState(
      proofHeight,
      proofAck,
      channel.counterpartyPortIdentifier,
      channel.counterpartyChannelIdentifier,
      expected
    ))
    channel.state = OPEN
    provableStore.set(channelPath(portIdentifier, channelIdentifier), channel)
}
```

##### 關閉握手

兩個模組中的任意一個透過調用`chanCloseInit`函數來關閉其通道端。一旦一端關閉，通道將無法重新打開。

調用模組可以在調用`chanCloseInit`時原子性的執行適當的應用程式邏輯。

通道關閉後，任何傳遞中的封包都會超時。

```typescript
function chanCloseInit(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
    abortTransactionUnless(authenticateCapability(channelCapabilityPath(portIdentifier, channelIdentifier), capability))
    channel = provableStore.get(channelPath(portIdentifier, channelIdentifier))
    abortTransactionUnless(channel !== null)
    abortTransactionUnless(channel.state !== CLOSED)
    connection = provableStore.get(connectionPath(channel.connectionHops[0]))
    abortTransactionUnless(connection !== null)
    abortTransactionUnless(connection.state === OPEN)
    channel.state = CLOSED
    provableStore.set(channelPath(portIdentifier, channelIdentifier), channel)
}
```

一旦一端已經關閉，對方模組調用`chanCloseConfirm`函數以關閉其通道端。

在調用`chanCloseConfirm`的同時，模組可以原子性的執行其他適當的應用邏輯。

關閉通道後，將無法重新打開通道，並且不能重複使用標識符。防止標識符重用是因為我們要防止潛在的重放先前發送的封包。重放問題類似於直接使用序號與帶簽名的消息，而不是用輕用戶端算法對消息（IBC 封包）進行“簽名”，防止重放的序列是埠標識符，通道標識符和封包序號的組合-因此我們不允許在序號重設為零的情況下，再次使用相同的埠標識符和通道標識符，因為這可能允許重放封包。如果要求並跟蹤特定最大高度/時間的超時，則可以安全的重用標識符，將來版本的規範可能包含此功能。

```typescript
function chanCloseConfirm(
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  proofInit: CommitmentProof,
  proofHeight: uint64) {
    abortTransactionUnless(authenticateCapability(channelCapabilityPath(portIdentifier, channelIdentifier), capability))
    channel = provableStore.get(channelPath(portIdentifier, channelIdentifier))
    abortTransactionUnless(channel !== null)
    abortTransactionUnless(channel.state !== CLOSED)
    connection = provableStore.get(connectionPath(channel.connectionHops[0]))
    abortTransactionUnless(connection !== null)
    abortTransactionUnless(connection.state === OPEN)
    expected = ChannelEnd{CLOSED, channel.order, portIdentifier,
                          channelIdentifier, [connection.counterpartyConnectionIdentifier], channel.version}
    abortTransactionUnless(connection.verifyChannelState(
      proofHeight,
      proofInit,
      channel.counterpartyPortIdentifier,
      channel.counterpartyChannelIdentifier,
      expected
    ))
    channel.state = CLOSED
    provableStore.set(channelPath(portIdentifier, channelIdentifier), channel)
}
```

#### 封包流和處理

![Packet State Machine](../../../../spec/ics-004-channel-and-packet-semantics/packet-state-machine.png)

##### 一個封包的日常

以下的步驟發生在一個封包從機器 *A* 上的模組 *1* 發送到機器 *B* 上的模組 *2*，從頭開始。

該模組可以通過 [ICS 25](../ics-025-handler-interface) 或 [ICS 26](../ics-026-routing-module) 接入 IBC 處理程序。

1. 以任何順序初始用戶端和埠設置
    1. 在 *A* 上為 *B* 創建用戶端（請參閱 [ICS 2](../ics-002-client-semantics) ）
    2. 在 *B* 上為 *A* 創建用戶端（請參閱 [ICS 2](../ics-002-client-semantics) ）
    3. 模組 *1* 綁定到埠（請參閱 [ICS 5](../ics-005-port-allocation) ）
    4. 模組 *2* 綁定到埠（請參閱 [ICS 5](../ics-005-port-allocation) ），該埠以帶外方式（out-of-band）傳輸到模組 *1*
2. 建立連接和通道，按順序樂觀發送（optimistic send）
    1. 模組 *1* 自 *A* 向 *B* 創建連接握手（請參見 [ICS 3](../ics-003-connection-semantics) ）
    2. 使用新創建的連接（此 ICS），自 *1* 向 *2* 開始創建通道握手
    3. 通過新創建的通道自 *1* 向 *2* 發送封包（此 ICS）
3. 握手成功完成（如果任一握手失敗，則連接/通道可以關閉且封包超時）
    1. 連接握手成功完成（請參閱 [ICS 3](../ics-003-connection-semantics) ）（這需要中繼器進程參與）
    2. 通道握手成功完成（此 ICS）（這需要中繼器進程的參與）
4. 在狀態機 *B* 的模組 *2* 上確認封包（如果超過超時區塊高度，則確認封包超時）（這將需要中繼器進程參與）
5. 確認消息從狀態機 *B* 上的模組 *2* 被中繼回狀態機 *A* 上的模組 *1*

從空間上表示，兩台機器之間的封包傳輸可以表示如下：

![Packet Transit](packet-transit.png)

##### 發送封包

`sendPacket`函數由模組調用，以便在調用模組的通道端將 IBC 封包發送到另一條鏈上的相應模組。

在調用`sendPacket`的同時，調用模組必須同時原子性的執行應用邏輯。

IBC 處理程序按順序執行以下步驟：

- 檢查用於發送封包的通道和連接是否打開
- 檢查調用模組是否擁有發送埠
- 檢查封包元數據與通道以及連接訊息是否匹配
- 檢查目標鏈尚未達到指定的超時區塊高度
- 遞增通道關聯的發送序號
- 存儲對封包數據和封包超時訊息的固定大小加密承諾

請注意，完整的封包不會存儲在鏈的狀態中——僅僅存儲數據和超時訊息的簡短哈希加密承諾。封包數據可以從交易的執行中計算得出，並可能作為中繼器可以索引的日誌輸出出來。

```typescript
function sendPacket(packet: Packet) {
    channel = provableStore.get(channelPath(packet.sourcePort, packet.sourceChannel))

    // optimistic sends are permitted once the handshake has started
    abortTransactionUnless(channel !== null)
    abortTransactionUnless(channel.state !== CLOSED)
    abortTransactionUnless(authenticateCapability(channelCapabilityPath(packet.sourcePort, packet.sourceChannel), capability))
    abortTransactionUnless(packet.destPort === channel.counterpartyPortIdentifier)
    abortTransactionUnless(packet.destChannel === channel.counterpartyChannelIdentifier)
    connection = provableStore.get(connectionPath(channel.connectionHops[0]))

    abortTransactionUnless(connection !== null)

    // sanity-check that the timeout height hasn't already passed in our local client tracking the receiving chain
    latestClientHeight = provableStore.get(clientPath(connection.clientIdentifier)).latestClientHeight()
    abortTransactionUnless(packet.timeoutHeight === 0 || latestClientHeight < packet.timeoutHeight)

    nextSequenceSend = provableStore.get(nextSequenceSendPath(packet.sourcePort, packet.sourceChannel))
    abortTransactionUnless(packet.sequence === nextSequenceSend)

    // all assertions passed, we can alter state

    nextSequenceSend = nextSequenceSend + 1
    provableStore.set(nextSequenceSendPath(packet.sourcePort, packet.sourceChannel), nextSequenceSend)
    provableStore.set(packetCommitmentPath(packet.sourcePort, packet.sourceChannel, packet.sequence),
                      hash(packet.data, packet.timeoutHeight, packet.timeoutTimestamp))

    // log that a packet has been sent
    emitLogEntry("sendPacket", {sequence: packet.sequence, data: packet.data, timeoutHeight: packet.timeoutHeight, timeoutTimestamp: packet.timeoutTimestamp})
}
```

#### 接受封包

模組調用`recvPacket`函數以接收和處理在對應的鏈的通道端發送的 IBC 封包。

在調用`recvPacket`函數的同時，調用模組必須原子性的執行應用邏輯，可能需要事先計算出封包確認消息的值。

IBC 處理程序按順序執行以下步驟：

- 檢查接收封包的通道和連接是否打開
- 檢查調用模組是否擁有接收埠
- 檢查封包元數據與通道及連接訊息是否匹配
- 檢查封包序號是通道端期望接收的（對於有序通道而言）
- 檢查尚未達到超時高度
- 在傳出鏈的狀態下檢查封包數據的加密承諾包含證明
- 在封包唯一的存儲路徑上設置一個不透明確認值（如果確認訊息為非空或是無序通道）
- 遞增與通道端關聯的封包接收序號（僅限有序通道）

```typescript
function recvPacket(
  packet: OpaquePacket,
  proof: CommitmentProof,
  proofHeight: uint64,
  acknowledgement: bytes): Packet {

    channel = provableStore.get(channelPath(packet.destPort, packet.destChannel))
    abortTransactionUnless(channel !== null)
    abortTransactionUnless(channel.state === OPEN)
    abortTransactionUnless(authenticateCapability(channelCapabilityPath(packet.destPort, packet.destChannel), capability))
    abortTransactionUnless(packet.sourcePort === channel.counterpartyPortIdentifier)
    abortTransactionUnless(packet.sourceChannel === channel.counterpartyChannelIdentifier)

    abortTransactionUnless(connection !== null)
    abortTransactionUnless(connection.state === OPEN)

    abortTransactionUnless(packet.timeoutHeight === 0 || getConsensusHeight() < packet.timeoutHeight)
    abortTransactionUnless(packet.timeoutTimestamp === 0 || currentTimestamp() < packet.timeoutTimestamp)

    abortTransactionUnless(connection.verifyPacketData(
      proofHeight,
      proof,
      packet.sourcePort,
      packet.sourceChannel,
      packet.sequence,
      concat(packet.data, packet.timeoutHeight, packet.timeoutTimestamp)
    ))

    // all assertions passed (except sequence check), we can alter state

    if (acknowledgement.length > 0 || channel.order === UNORDERED)
      provableStore.set(
        packetAcknowledgementPath(packet.destPort, packet.destChannel, packet.sequence),
        hash(acknowledgement)
      )

    if (channel.order === ORDERED) {
      nextSequenceRecv = provableStore.get(nextSequenceRecvPath(packet.destPort, packet.destChannel))
      abortTransactionUnless(packet.sequence === nextSequenceRecv)
      nextSequenceRecv = nextSequenceRecv + 1
      provableStore.set(nextSequenceRecvPath(packet.destPort, packet.destChannel), nextSequenceRecv)
    } else {
      abortTransactionUnless(provableStore.get(packetAcknowledgementPath(packet.destPort, packet.destChannel, packet.sequence) === null))
    }

    // log that a packet has been received & acknowledged
    emitLogEntry("recvPacket", {sequence: packet.sequence, timeoutHeight: packet.timeoutHeight,
                                timeoutTimestamp: packet.timeoutTimestamp, data: packet.data, acknowledgement})

    // return transparent packet
    return packet
}
```

#### 確認

模組會調用`acknowledgePacket`函數來處理先前由對方鏈上對方模組的通道上的調用模組發送的封包的確認。 `acknowledgePacket`還會清除封包的加密承諾，因為封包已經收到並處理，所以這個不再需要。

在調用`acknowledgePacket`的同時，調用模組可以原子性的執行適當的應用程式確認處理邏輯。

```typescript
function acknowledgePacket(
  packet: OpaquePacket,
  acknowledgement: bytes,
  proof: CommitmentProof,
  proofHeight: uint64): Packet {

    // abort transaction unless that channel is open, calling module owns the associated port, and the packet fields match
    channel = provableStore.get(channelPath(packet.sourcePort, packet.sourceChannel))
    abortTransactionUnless(channel !== null)
    abortTransactionUnless(channel.state === OPEN)
    abortTransactionUnless(authenticateCapability(channelCapabilityPath(packet.sourcePort, packet.sourceChannel), capability))
    abortTransactionUnless(packet.destPort === channel.counterpartyPortIdentifier)
    abortTransactionUnless(packet.destChannel === channel.counterpartyChannelIdentifier)

    connection = provableStore.get(connectionPath(channel.connectionHops[0]))
    abortTransactionUnless(connection !== null)
    abortTransactionUnless(connection.state === OPEN)

    // verify we sent the packet and haven't cleared it out yet
    abortTransactionUnless(provableStore.get(packetCommitmentPath(packet.sourcePort, packet.sourceChannel, packet.sequence))
           === hash(packet.data, packet.timeoutHeight, packet.timeoutTimestamp))

    // abort transaction unless correct acknowledgement on counterparty chain
    abortTransactionUnless(connection.verifyPacketAcknowledgement(
      proofHeight,
      proof,
      packet.destPort,
      packet.destChannel,
      packet.sequence,
      acknowledgement
    ))

    // abort transaction unless acknowledgement is processed in order
    if (channel.order === ORDERED) {
      nextSequenceAck = provableStore.get(nextSequenceAckPath(packet.sourcePort, packet.sourceChannel))
      abortTransactionUnless(packet.sequence === nextSequenceAck)
      nextSequenceAck = nextSequenceAck + 1
      provableStore.set(nextSequenceAckPath(packet.sourcePort, packet.sourceChannel), nextSequenceAck)
    }

    // all assertions passed, we can alter state

    // delete our commitment so we can't "acknowledge" again
    provableStore.delete(packetCommitmentPath(packet.sourcePort, packet.sourceChannel, packet.sequence))

    // return transparent packet
    return packet
}
```

#### 超時

應用程式語義可能需要定義超時：超時是鏈在將一筆交易視作錯誤之前將等待多長時間的上限。由於這兩個鏈本地時間的不同，因此這是一個明顯的雙花攻擊的方向——攻擊者可能延遲發送確認消息或在超時時間後發送封包——因此應用程式本身無法安全的實現簡單的超時邏輯。

請注意，為了避免任何可能的“雙花”攻擊，超時算法要求目標鏈正在運行並且是可訪問的。一個人不能在一個網路完全分區（network parition）的情況下證明任何事情，並且必須等待連接。必須在接收者鏈上證明已經超時，而不僅僅是以發送鏈沒有收到響應作為判斷。

##### 發送端

`timeoutPacket`函數由最初嘗試將封包發送到對方鏈的模組在沒有提交封包的情況下對方鏈達到超時區塊高度或超過超時時間戳的情況下調用，以證明該封包無法再執行，並允許調用模組安全的執行適當的狀態轉換。

在調用`timeoutPacket`的同時，調用模組可以原子性的執行適當的應用超時處理邏輯。

在有序通道的情況下，`timeoutPacket`檢查接收通道端的`recvSequence` ，如果封包已超時，則關閉通道。

在無序通道的情況下， `timeoutPacket`檢查是否存在確認（如果接收到封包，則該確認將被寫入）。面對超時的封包，無序通道預期會繼續工作。

如果連續的封包的超時高度之間是強制的關係，則可以執行所有封包的安全批次超時而不是使用超時封包。該規範暫時省略了細節。

```typescript
function timeoutPacket(
  packet: OpaquePacket,
  proof: CommitmentProof,
  proofHeight: uint64,
  nextSequenceRecv: Maybe<uint64>): Packet {

    channel = provableStore.get(channelPath(packet.sourcePort, packet.sourceChannel))
    abortTransactionUnless(channel !== null)
    abortTransactionUnless(channel.state === OPEN)

    abortTransactionUnless(authenticateCapability(channelCapabilityPath(packet.sourcePort, packet.sourceChannel), capability))
    abortTransactionUnless(packet.destChannel === channel.counterpartyChannelIdentifier)

    connection = provableStore.get(connectionPath(channel.connectionHops[0]))
    // note: the connection may have been closed
    abortTransactionUnless(packet.destPort === channel.counterpartyPortIdentifier)

    // check that timeout height or timeout timestamp has passed on the other end
    abortTransactionUnless(
      (packet.timeoutHeight > 0 && proofHeight >= packet.timeoutHeight) ||
      (packet.timeoutTimestamp > 0 && connection.getTimestampAtHeight(proofHeight) > packet.timeoutTimestamp))

    // verify we actually sent this packet, check the store
    abortTransactionUnless(provableStore.get(packetCommitmentPath(packet.sourcePort, packet.sourceChannel, packet.sequence))
           === hash(packet.data, packet.timeoutHeight, packet.timeoutTimestamp))

    if channel.order === ORDERED {
      // ordered channel: check that packet has not been received
      abortTransactionUnless(nextSequenceRecv <= packet.sequence)
      // ordered channel: check that the recv sequence is as claimed
      abortTransactionUnless(connection.verifyNextSequenceRecv(
        proofHeight,
        proof,
        packet.destPort,
        packet.destChannel,
        nextSequenceRecv
      ))
    } else
      // unordered channel: verify absence of acknowledgement at packet index
      abortTransactionUnless(connection.verifyPacketAcknowledgementAbsence(
        proofHeight,
        proof,
        packet.destPort,
        packet.destChannel,
        packet.sequence
      ))

    // all assertions passed, we can alter state

    // delete our commitment
    provableStore.delete(packetCommitmentPath(packet.sourcePort, packet.sourceChannel, packet.sequence))

    if channel.order === ORDERED {
      // ordered channel: close the channel
      channel.state = CLOSED
      provableStore.set(channelPath(packet.sourcePort, packet.sourceChannel), channel)
    }

    // return transparent packet
    return packet
}
```

##### 關閉時超時

模組調用`timeoutOnClose`函數，以證明一個未接收到封包所定址的通道已關閉，因此將永遠不會接收到該封包（即使尚未達到`timeoutHeight`或`timeoutTimestamp` ）。

```typescript
function timeoutOnClose(
  packet: Packet,
  proof: CommitmentProof,
  proofClosed: CommitmentProof,
  proofHeight: uint64,
  nextSequenceRecv: Maybe<uint64>): Packet {

    channel = provableStore.get(channelPath(packet.sourcePort, packet.sourceChannel))
    // note: the channel may have been closed
    abortTransactionUnless(authenticateCapability(channelCapabilityPath(packet.sourcePort, packet.sourceChannel), capability))
    abortTransactionUnless(packet.destChannel === channel.counterpartyChannelIdentifier)

    connection = provableStore.get(connectionPath(channel.connectionHops[0]))
    // note: the connection may have been closed
    abortTransactionUnless(packet.destPort === channel.counterpartyPortIdentifier)

    // verify we actually sent this packet, check the store
    abortTransactionUnless(provableStore.get(packetCommitmentPath(packet.sourcePort, packet.sourceChannel, packet.sequence))
           === hash(packet.data, packet.timeoutHeight, packet.timeoutTimestamp))

    // check that the opposing channel end has closed
    expected = ChannelEnd{CLOSED, channel.order, channel.portIdentifier,
                          channel.channelIdentifier, channel.connectionHops.reverse(), channel.version}
    abortTransactionUnless(connection.verifyChannelState(
      proofHeight,
      proofClosed,
      channel.counterpartyPortIdentifier,
      channel.counterpartyChannelIdentifier,
      expected
    ))

    if channel.order === ORDERED {
      // ordered channel: check that the recv sequence is as claimed
      abortTransactionUnless(connection.verifyNextSequenceRecv(
        proofHeight,
        proof,
        packet.destPort,
        packet.destChannel,
        nextSequenceRecv
      ))
      // ordered channel: check that packet has not been received
      abortTransactionUnless(nextSequenceRecv <= packet.sequence)
    } else
      // unordered channel: verify absence of acknowledgement at packet index
      abortTransactionUnless(connection.verifyPacketAcknowledgementAbsence(
        proofHeight,
        proof,
        packet.destPort,
        packet.destChannel,
        packet.sequence
      ))

    // all assertions passed, we can alter state

    // delete our commitment
    provableStore.delete(packetCommitmentPath(packet.sourcePort, packet.sourceChannel, packet.sequence))

    // return transparent packet
    return packet
}
```

##### 清理狀態

模組調用`cleanupPacket`以從狀態中刪除收到的封包承諾。接收端必須已經處理過該封包（無論是正常處理還是超時）。

在有序通道的情況下， `cleanupPacket`透過證明已在另一端收到封包來清理有序通道上的封包。

在無序通道的情況下， `cleanupPacket`通過證明已寫入關聯的確認訊息清理無序通道上的封包。

```typescript
function cleanupPacket(
  packet: OpaquePacket,
  proof: CommitmentProof,
  proofHeight: uint64,
  nextSequenceRecvOrAcknowledgement: Either<uint64, bytes>): Packet {

    channel = provableStore.get(channelPath(packet.sourcePort, packet.sourceChannel))
    abortTransactionUnless(channel !== null)
    abortTransactionUnless(channel.state === OPEN)
    abortTransactionUnless(authenticateCapability(channelCapabilityPath(packet.sourcePort, packet.sourceChannel), capability))
    abortTransactionUnless(packet.destChannel === channel.counterpartyChannelIdentifier)

    connection = provableStore.get(connectionPath(channel.connectionHops[0]))
    abortTransactionUnless(connection !== null)
    // note: the connection may have been closed
    abortTransactionUnless(packet.destPort === channel.counterpartyPortIdentifier)

    // abortTransactionUnless packet has been received on the other end
    abortTransactionUnless(nextSequenceRecv > packet.sequence)

    // verify we actually sent the packet, check the store
    abortTransactionUnless(provableStore.get(packetCommitmentPath(packet.sourcePort, packet.sourceChannel, packet.sequence))
               === hash(packet.data, packet.timeoutHeight, packet.timeoutTimestamp))

    if channel.order === ORDERED
      // check that the recv sequence is as claimed
      abortTransactionUnless(connection.verifyNextSequenceRecv(
        proofHeight,
        proof,
        packet.destPort,
        packet.destChannel,
        nextSequenceRecvOrAcknowledgement
      ))
    else
      // abort transaction unless acknowledgement on the other end
      abortTransactionUnless(connection.verifyPacketAcknowledgement(
        proofHeight,
        proof,
        packet.destPort,
        packet.destChannel,
        packet.sequence,
        nextSequenceRecvOrAcknowledgement
      ))

    // all assertions passed, we can alter state

    // clear the store
    provableStore.delete(packetCommitmentPath(packet.sourcePort, packet.sourceChannel, packet.sequence))

    // return transparent packet
    return packet
}
```

#### 關於競態條件的探討

##### 同時發生握手嘗試

如果兩台狀態機同時彼此發起通道創建握手，或嘗試使用相同的標識符，則兩者都會失敗，必須使用新的標識符。

##### 標識符分配

在目標鏈上分配標識符存在不可避免的競態條件。最好建議模組使用偽隨機，無價值的標識符。設法聲明另一個模組希望使用的標識符，但是，儘管令人煩惱，中間人卻無法在握手期間攻擊，因為接收模組必須已經擁有握手所針對的埠。

##### 超時/封包確認

封包超時和封包確認之間沒有競態條件，因為封包只能在接收之前檢查超過或沒超過超時區塊高度。

##### 握手期間的中間人攻擊

跨鏈狀態的驗證可防止連接握手和通道握手期間的中間人攻擊，因為模組已知道所有訊息（源用戶端、目標用戶端、通道等），該訊息將在啟動握手之前進行確認完成。

##### 有正在傳輸封包時的連接/通道關閉

如果在傳輸封包時關閉了連接或通道，則封包將不再被目標連結收，並且在源鏈上超時。

#### 查詢通道

可以使用`queryChannel`函數查詢通道：

```typescript
function queryChannel(connId: Identifier, chanId: Identifier): ChannelEnd | void {
    return provableStore.get(channelPath(connId, chanId))
}
```

### 屬性和不變數

- 通道和埠標識符的唯一組合是先到先服務的：分配了一對標示符後，只有擁有相應埠的模組才能在該通道上發送或接收。
- 假設鏈在超時窗口後依然有活性，則封包只傳遞一次，並且在超時的情況下，只在發送鏈上超時一次。
- 通道握手不能受到區塊鏈上的另一個模組或另一個區塊鏈的 IBC 處理程序的中間人攻擊。

## 向後相容性

不適用。

## 向前相容性

數據結構和編碼可以在連接或通道級別進行版本控制。通道邏輯完全不依賴於封包數據格式，可以由模組在任何時候以自己喜歡的方式對其進行更改。

## 範例實現

即將到來。

## 其他實現

即將到來。

## 發布歷史

2019年6月5日-提交草案

2019年7月4日-修改無序通道和確認

2019年7月16日-更改“多跳”路由未來的相容性

2019年7月29日-修改以處理連接關閉後的超時

2019年8月13日-各種修改

2019年8月25日-清理

## 版權

本文中的所有內容均根據 [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0) 獲得許可。
