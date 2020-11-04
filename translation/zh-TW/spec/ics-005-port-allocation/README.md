---
ics: 5
title: 埠分配
stage: 草案
requires: 24
required-by: 4
category: IBC/TAO
kind: 介面
author: Christopher Goes <cwgoes@tendermint.com>
created: 2019-06-20
modified: 2019-08-25
---

## 概要

該標準指定了埠分配系統，模組可以通過該系統綁定到由 IBC 處理程序分配的唯一命名的埠。 然後可以將埠用於創建通道，並且可以被最初綁定到埠的模組轉移或釋放。

### 動機

區塊鏈間通信協議旨在促進模組之間的通信，其中模組是獨立的，可能相互不信任，在自治帳本上執行的自成一體的代碼。為了提供所需的端到端語義，IBC 處理程序必須實現對特定模組許可的通道。 該規範定義了實現該模型的*埠分配和所有權*系統。

可能會出現關於哪種模組邏輯可以綁定到特定埠名稱的約定，例如“bank”用於處理同質通證，“staking”用於鏈間抵押的。 這類似於 HTTP 伺服器的 80 埠的慣用用法——該協議無法強制將特定的模組邏輯實際上綁定到慣用埠，因此用戶必須自己檢查。可以創建具有偽隨機標識符的臨時埠以用於臨時協議處理。

模組可以綁定到多個埠，並連接到單獨計算機上另一個模組綁定的多個埠。任何數量的（唯一標識的）通道都可以同時使用一個埠。通道是在兩個埠之間的端到端的，每個埠必須事先已被模組綁定，然後模組將控制該通道的一端。

（可選）主機狀態機可以選擇將埠綁定透過生成專門用於綁定埠的能力鍵的方式暴露給特別允許的模組管理器 。然後模組管理器可以使用自訂規則集控制模組可以綁定到哪些埠，和轉移埠到僅已驗證埠名稱和模組的模組。路由模組可以扮演這個角色（請參閱 [ICS 26](../ics-026-routing-module) ）。

### 定義

`Identifier` ， `get` ， `set`和`delete`的定義與 [ICS 24](../ics-024-host-requirements) 中的相同。

*埠*是一種特殊的標識符，用於許可模組創建和使用通道。

*模組*是主機狀態機的子組件，獨立於 IBC 處理程序。範例包括以太坊智慧合約和  Cosmos SDK 和 Substrate 的模組。 除了主機狀態機可以使用對象能力或源身份驗證來訪問模組的許可埠的能力之外，IBC 規範不對模組功能進行任何假設。

### 所需屬性

- 一個模組綁定到埠後，其他模組將無法使用該埠，直到該模組釋放它
- 一個模組可以選擇釋放埠或將其轉移到另一個模組
- 單個模組可以一次綁定到多個埠
- 分配埠時，先到先得，先綁定先服務，鏈可以在第一次啟動時將已知模組綁定“保留”埠。

作為一個有幫助的比較，以下 TCP 的類比大致準確：

IBC 概念 | TCP/IP 概念 | 差異性
--- | --- | ---
IBC | TCP | 很多，請參閱描述 IBC 的體系結構文件
埠（例如“bank”） | 埠（例如 80） | 沒有數字較小的保留埠，埠為字串
模組（例如“bank”） | 應用程式（例如 Nginx） | 特定於應用
用戶端 | - | 沒有直接的類比，有點像 L2 路由，也有點像 TLS
連接 | - | 沒有直接的類比，合併進了 TCP 的連接
通道 | 連接 | 可以同時打開或關閉任意數量的通道

## 技術指標

### 數據結構

主機狀態機務必支持對象能力引用或模組的源認證。

在前一種支持對象能力的情況下，IBC 處理程序必須支持生成唯一的，不透明的*對象能力*引用的能力，它可以傳遞給某個模組，而其他模組則無法複製。兩個範例是 Cosmos SDK（ [參考](https://github.com/cosmos/cosmos-sdk/blob/97eac176a5d533838333f7212cbbd79beb0754bc/store/types/store.go#L275) ）中使用的存儲金鑰和 Agoric 的 Javascript 運行時中使用的對象引用（ [參考](https://github.com/Agoric/SwingSet) ）。

```typescript
type CapabilityKey object
```

`newCapability`必須使用一個名稱生成唯一的能力鍵，這樣該名稱將本地映射到能力鍵，並且以後可以與`getCapability`一起使用。

```typescript
function newCapability(name: string): CapabilityKey {
  // provided by host state machine, e.g. ADR 3 / ScopedCapabilityKeeper in Cosmos SDK
}
```

`authenticateCapability`必須接受一個名稱和能力鍵，並檢查名稱是否在本地映射到所提供的能力。該名稱可以是不受信任的用戶輸入。

```typescript
function authenticateCapability(name: string, capability: CapabilityKey): bool {
  // provided by host state machine, e.g. ADR 3 / ScopedCapabilityKeeper in Cosmos SDK
}
```

`claimCapability`必須接受一個名稱和能力鍵（由另一個模組提供），並將其本地映射到能力，“聲明”該能力以備將來使用。

```typescript
function claimCapability(name: string, capability: CapabilityKey) {
  // provided by host state machine, e.g. ADR 3 / ScopedCapabilityKeeper in Cosmos SDK
}
```

`getCapability`必須允許模組查找其先前創建的能力或按名稱聲明的能力。

```typescript
function getCapability(name: string): CapabilityKey {
  // provided by host state machine, e.g. ADR 3 / ScopedCapabilityKeeper in Cosmos SDK
}
```

`releaseCapability`必須允許模組釋放其擁有的能力。

```typescript
function releaseCapability(capability: CapabilityKey) {
  // provided by host state machine, e.g. ADR 3 / ScopedCapabilityKeeper in Cosmos SDK
}
```

在後一種源身份驗證的情況下，IBC 處理程序必須具有安全讀取調用模組的*源標識符*的能力， 主機狀態機中每個模組的唯一字串，不能由該模組更改或由另一個模組偽造。 一個範例是以太坊（ [參考](https://ethereum.github.io/yellowpaper/paper.pdf) ）使用的智慧合約地址。

```typescript
type SourceIdentifier string
```

```typescript
function callingModuleIdentifier(): SourceIdentifier {
  // provided by host state machine, e.g. contract address in Ethereum
}
```

然後按以下方式實現`newCapability` ， `authenticateCapability` ， `claimCapability` ， `getCapability`和`releaseCapability` ：

```
function newCapability(name: string): CapabilityKey {
  return callingModuleIdentifier()
}
```

```
function authenticateCapability(name: string, capability: CapabilityKey) {
  return callingModuleIdentifier() === name
}
```

```
function claimCapability(name: string, capability: CapabilityKey) {
  // no-op
}
```

```
function getCapability(name: string): CapabilityKey {
  // not actually used
  return nil
}
```

```
function releaseCapability(capability: CapabilityKey) {
  // no-op
}
```

#### 儲存路徑

`portPath`接受一個`Identifier`參數並返回存儲路徑，在該路徑下應存儲與埠關聯的對象能力引用或所有者模組標識符。

```typescript
function portPath(id: Identifier): Path {
    return "ports/{id}"
}
```

### 子協議

#### 標識符驗證

埠的所有者模組標識符存儲在唯一的`Identifier`前綴下。 可以提供驗證函數`validatePortIdentifier` 。

```typescript
type validatePortIdentifier = (id: Identifier) => boolean
```

如果未提供，預設的`validatePortIdentifier`函數將始終返回`true` 。

#### 綁定到埠

IBC 處理程序必須實現`bindPort` 。 `bindPort`綁定到未分配的埠，如果該埠已被分配，則綁定失敗。

如果主機狀態機未實現特殊的模組管理器來控制埠分配，則`bindPort`應該對所有模組都可用。否則`bindPort`應該只能由模組管理器調用。

```typescript
function bindPort(id: Identifier): CapabilityKey {
    abortTransactionUnless(validatePortIdentifier(id))
    abortTransactionUnless(getCapability(portPath(id)) === null)
    capability = newCapability(portPath(id))
    return capability
}
```

#### 轉讓埠所有權

如果主機狀態機支持對象能力，則不需要增加此協議，因為埠的引用承載了能力。

#### 釋放埠

IBC 處理程序必須實現`releasePort`函數，該函數允許模組釋放埠，以便其他模組隨後可以綁定到該埠。

`releasePort`應該對所有模組都可用。

> 警告：釋放埠將允許其他模組綁定到該埠，並可能攔截傳入的通道創建握手請求。僅在安全的情況下，模組才應釋放埠。

```typescript
function releasePort(capability: CapabilityKey) {
    abortTransactionUnless(authenticateCapability(portPath(id), capability))
    releaseCapability(capability)
}
```

### 屬性和不變數

- 默認情況下，埠標識符是先到先服務的：模組綁定到埠後，只有該模組才能使用該埠，直到模組轉移或釋放它為止。模組管理器可以實現自訂邏輯，以覆蓋此邏輯。

## 向後相容性

不適用。

## 向前相容性

埠綁定不是線協議（wire protocol），因此只要所有權語義不受影響，介面就可以在單獨的鏈上獨立更改。

## 範例實現

即將到來。

## 其他實現

即將到來。

## 歷史

2019年6月29日-初稿

## 版權

本文中的所有內容均根據 [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0) 獲得許可。
