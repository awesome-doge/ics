---
ics: 25
title: 處理程序介面
stage: 草案
category: IBC/TAO
kind: 實例化
requires: 2, 3, 4, 23, 24
author: Christopher Goes <cwgoes@tendermint.com>
created: 2019-04-23
modified: 2019-08-25
---

## 概要

本文件描述了標準 IBC 實現（稱為 IBC 處理程序）向同一狀態機內的模組公開的介面，以及 IBC 處理程序對該介面的實現。

### 動機

IBC 是一種模組間通信協議，旨在促進可靠的，經過身份驗證的消息在獨立的區塊鏈上的模組之間傳遞。模組應該能夠推理出與之交互的介面以及為了安全的使用介面而必須遵守的要求。

### 定義

相關定義與相應的先前標準（定義了函數）中的定義相同。

### 所需屬性

- 用戶端，連接和通道的創建應儘可能的不需許可。
- 模組集應該是動態的：鏈應該能夠添加和刪除模組，這些模組本身可以使用持久性 IBC 處理程序任意綁定到埠或從埠取消綁定。
- 模組應該能夠在 IBC 之上編寫自己的更複雜的抽象，以提供附加的語義或保證。

## 技術指標

> 注意：如果主機狀態機正在使用對象能力認證（請參閱 [ICS 005](../ics-005-port-allocation) ），則所有使用埠的函數都將帶有附加的能力鍵參數。

### 用戶端生命週期管理

默認情況下，用戶端是沒有所有者的：任何模組都可以創建新用戶端，查詢任何現有用戶端，更新任何現有用戶端以及刪除任何未使用的現有用戶端。

處理程序介面暴露 [ICS 2](../ics-002-client-semantics) 中定義的`createClient` ， `updateClient` ， `queryClientConsensusState` ， `queryClient`和`submitMisbehaviourToClient` 。

### 連接生命週期管理

處理程序介面暴露 [ICS 3](../ics-003-connection-semantics) 中定義的`connOpenInit` ， `connOpenTry` ， `connOpenAck` ， `connOpenConfirm`和`queryConnection` 。

默認的 IBC 路由模組應允許外部調用`connOpenTry` ， `connOpenAck`和`connOpenConfirm` 。

### 通道生命週期管理

默認情況下，通道歸創建的埠所有，這意味著只允許綁定到該埠的模組檢視，關閉或在通道上發送。模組可以使用同一埠創建任意數量的通道。

處理程序介面暴露了 [ICS 4](../ics-004-channel-and-packet-semantics) 中定義的`chanOpenInit` ， `chanOpenTry` ， `chanOpenAck` ， `chanOpenConfirm` ， `chanCloseInit` ， `chanCloseConfirm`和`queryChannel` 。

默認的 IBC 路由模組應允許外部調用`chanOpenTry` ， `chanOpenAck` ， `chanOpenConfirm`和`chanCloseConfirm` 。

### 封包中繼

封包是需要通道許可的（只有擁有通道的埠可以在其上發送或接收）。

該處理程序介面暴露`sendPacket` ， `recvPacket` ， `acknowledgePacket` ， `timeoutPacket` ， `timeoutOnClose`和`cleanupPacket`，如 [ICS 4](../ics-004-channel-and-packet-semantics)中定義 。

默認  IBC 路由模組應允許外部調用`sendPacket` ， `recvPacket` ， `acknowledgePacket` ， `timeoutPacket` ， `timeoutOnClose`和`cleanupPacket` 。

### 屬性和不變數

此處定義的 IBC 處理程序模組介面繼承了其關聯規範中定義的功能屬性。

## 向後相容性

不適用。

## 向前相容性

只要在語義上相同，在新鏈上實現（或升級到現有鏈）時，此介面可以更改。

## 範例實現

即將到來。

## 其他實現

即將到來。

## 歷史

2019年6月9日-編寫草案

2019年8月24日-修訂，清理

## 版權

本文中的所有內容均根據 [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0) 獲得許可。
