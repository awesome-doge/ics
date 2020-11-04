---
ics: 18
title: 中繼器算法
stage: 草案
category: IBC/TAO
kind: 介面
requires: 24, 25, 26
author: Christopher Goes <cwgoes@tendermint.com>
created: 2019-03-07
modified: 2019-08-25
---

## 概要

中繼器算法是 IBC 的“物理”連接層-鏈下進程通過掃描鏈的狀態，構造適當的數據報，並按照 IBC 規定在對方鏈上執行，從而在運行 IBC 協議的兩條鏈之間中繼數據。

### 動機

在 IBC 協議中，區塊鏈只能記錄將特定數據發送到另一條鏈的*意圖* -它不能直接訪問網路傳輸層。物理數據報的中繼必須由可訪問傳輸層（例如 TCP/IP）的鏈下基礎設施執行。該標準定義了*中繼器*算法，可以查詢鏈狀態的鏈下進程執行這個算法，來實現中繼。

### 定義

*中繼器*是一種鏈下進程，能夠使用 IBC 協議讀取狀態並將交易提交到某些帳本集。

### 所需屬性

- IBC 的僅一次傳遞或超時安全屬性都不應依賴中繼器的行為（假設中繼器可以有拜占庭行為）。
- IBC 的中繼活性僅應依賴於至少一個正確的，活躍的中繼器存在。
- 中繼應該是不需許可的，所有必要的驗證都應在鏈上執行。
- 應該最小化 IBC 用戶和中繼器之間的必要通信。
- 應能在應用層提供中繼器激勵措施。

## 技術指標

### 基礎中繼器算法

中繼器算法是在一個實現了 IBC 協議的鏈集`C`上定義的。每個中繼器不一定需要訪問鏈間網路中所有鏈的狀態來讀取數據報或將數據報寫入鏈間網路中的所有鏈（尤其是在許可鏈或私有鏈的情況下），不同的中繼器可以在不同子集之間中繼。

`pendingDatagrams`根據兩條鏈的狀態計算要從一個鏈中繼到另一個鏈的所有有效數據報的集合。中繼器必須具有為其中繼的集合中的區塊鏈實現了哪些 IBC 協議的子集的先驗知識（例如，通過閱讀原始碼）。下面定義了一個範例。

`submitDatagram`是鏈自己定義的過程（提交某個交易）。數據報可以每個當作單獨的交易提交，也可以在鏈支持的情況下作為一整個交易原子性提交。

`relay`每隔一段時間就會調用一次 - 不高於任一鏈的出塊速度，並且可能根據中繼器期望的中繼頻率而降低一些。

不同的中繼器可以在不同的鏈之間進行中繼-只要每對鏈具有至少一個正確且活躍的中繼器，這些鏈就可以保持活性，網路中鏈之間流動的所有封包最終都將被中繼。

```typescript
function relay(C: Set<Chain>) {
  for (const chain of C)
    for (const counterparty of C)
      if (counterparty !== chain) {
        const datagrams = chain.pendingDatagrams(counterparty)
        for (const localDatagram of datagrams[0])
          chain.submitDatagram(localDatagram)
        for (const counterpartyDatagram of datagrams[1])
          counterparty.submitDatagram(counterpartyDatagram)
      }
}
```

### 封包，確認，超時

#### 在有序通道中中繼封包

可以基於事件的方式或基於查詢的方式中繼有序通道中的封包。對於前者，中繼器應監視源鏈，每當發送封包發出事件時，使用事件日誌中的數據來組成封包。對於後者，中繼器應定期查詢源鏈上的發送序號，並保持中繼的最後一個序號，兩者之間的任何序號都是需要查詢然後中繼的封包。無論哪種情況，中繼器進程都應通過檢查接收序號來檢查目的鏈是否尚未接收到這個封包，然後才進行中繼。

#### 在無序通道中中繼封包

可以基於事件的方式中繼無序通道中的封包。中繼器應監視源鏈中每個發送封包發出的事件，然後使用事件日誌中的數據來組成封包。隨後，中繼器應透過查詢封包的序號是否存在對應的確認來檢查目的鏈是否已接收到過該封包，如果尚未出現，中繼器才中繼該封包。

#### 中繼確認

確認可以基於事件的方式進行中繼。中繼器應該監視目標鏈，每當接收封包並寫入確認發出事件時，使用事件日誌中的數據組成確認封包，檢查封包承諾在源鏈上是否存在（一旦確認被中繼，它將被刪除），如果是，則將確認中繼到源鏈。

#### 中繼超時

超時中繼稍微複雜一些，因為當封包超時時沒有特定事件發出，這是簡單的情況，由於目標鏈已經超過超時高度或時間戳，因此無法再中繼封包。中繼器進程必須選擇跟蹤一組封包（可以透過掃描事件日誌來構造），並且一旦目的鏈的高度或時間戳超過跟蹤的封包的高度或時間戳，就檢查封包承諾是否仍存在於源鏈（一旦超時被中繼，它將被刪除），如果是，則將超時中繼到源鏈。

### 待處理的數據報

`pendingDatagrams`整理要從一台機器發送到另一台機器的數據報。此功能的實現將取決於兩台機器都支持的 IBC 協議的子集以及源機器的狀態布局。特定的中繼器可能還會實現其自己的過濾器功能，以便僅中繼可被中繼的數據報的子集（例如，一個為了能中繼而鏈下付過費的子集）。

下面概述了在兩個鏈之間執行單向中繼的範例實現。通過交換`chain`和`counterparty` ，可以更改為執行雙向中繼。 哪個中繼器進程負責哪個數據報是一個靈活的選擇-在此範例中，中繼器進程中繼在`chain`上開始的所有握手（將數據報發送到兩個鏈），中繼從`chain`發送的所有封包到`counterparty` ，並中繼所有封包的確認從`counterparty`發送到`chain` 。

```typescript
function pendingDatagrams(chain: Chain, counterparty: Chain): List<Set<Datagram>> {
  const localDatagrams = []
  const counterpartyDatagrams = []

  // ICS2 : Clients
  // - Determine if light client needs to be updated (local & counterparty)
  height = chain.latestHeight()
  client = counterparty.queryClientConsensusState(chain)
  if client.height < height {
    header = chain.latestHeader()
    counterpartyDatagrams.push(ClientUpdate{chain, header})
  }
  counterpartyHeight = counterparty.latestHeight()
  client = chain.queryClientConsensusState(counterparty)
  if client.height < counterpartyHeight {
    header = counterparty.latestHeader()
    localDatagrams.push(ClientUpdate{counterparty, header})
  }

  // ICS3 : Connections
  // - Determine if any connection handshakes are in progress
  connections = chain.getConnectionsUsingClient(counterparty)
  for (const localEnd of connections) {
    remoteEnd = counterparty.getConnection(localEnd.counterpartyIdentifier)
    if (localEnd.state === INIT && remoteEnd === null)
      // Handshake has started locally (1 step done), relay `connOpenTry` to the remote end
      counterpartyDatagrams.push(ConnOpenTry{
        desiredIdentifier: localEnd.counterpartyConnectionIdentifier,
        counterpartyConnectionIdentifier: localEnd.identifier,
        counterpartyClientIdentifier: localEnd.clientIdentifier,
        counterpartyPrefix: localEnd.commitmentPrefix,
        clientIdentifier: localEnd.counterpartyClientIdentifier,
        version: localEnd.version,
        counterpartyVersion: localEnd.version,
        proofInit: localEnd.proof(),
        proofConsensus: localEnd.client.consensusState.proof(),
        proofHeight: height,
        consensusHeight: localEnd.client.height,
      })
    else if (localEnd.state === INIT && remoteEnd.state === TRYOPEN)
      // Handshake has started on the other end (2 steps done), relay `connOpenAck` to the local end
      localDatagrams.push(ConnOpenAck{
        identifier: localEnd.identifier,
        version: remoteEnd.version,
        proofTry: remoteEnd.proof(),
        proofConsensus: remoteEnd.client.consensusState.proof(),
        proofHeight: remoteEnd.client.height,
        consensusHeight: remoteEnd.client.height,
      })
    else if (localEnd.state === OPEN && remoteEnd.state === TRYOPEN)
      // Handshake has confirmed locally (3 steps done), relay `connOpenConfirm` to the remote end
      counterpartyDatagrams.push(ConnOpenConfirm{
        identifier: remoteEnd.identifier,
        proofAck: localEnd.proof(),
        proofHeight: height,
      })
  }

  // ICS4 : Channels & Packets
  // - Determine if any channel handshakes are in progress
  // - Determine if any packets, acknowledgements, or timeouts need to be relayed
  channels = chain.getChannelsUsingConnections(connections)
  for (const localEnd of channels) {
    remoteEnd = counterparty.getConnection(localEnd.counterpartyIdentifier)
    // Deal with handshakes in progress
    if (localEnd.state === INIT && remoteEnd === null)
      // Handshake has started locally (1 step done), relay `chanOpenTry` to the remote end
      counterpartyDatagrams.push(ChanOpenTry{
        order: localEnd.order,
        connectionHops: localEnd.connectionHops.reverse(),
        portIdentifier: localEnd.counterpartyPortIdentifier,
        channelIdentifier: localEnd.counterpartyChannelIdentifier,
        counterpartyPortIdentifier: localEnd.portIdentifier,
        counterpartyChannelIdentifier: localEnd.channelIdentifier,
        version: localEnd.version,
        counterpartyVersion: localEnd.version,
        proofInit: localEnd.proof(),
        proofHeight: height,
      })
    else if (localEnd.state === INIT && remoteEnd.state === TRYOPEN)
      // Handshake has started on the other end (2 steps done), relay `chanOpenAck` to the local end
      localDatagrams.push(ChanOpenAck{
        portIdentifier: localEnd.portIdentifier,
        channelIdentifier: localEnd.channelIdentifier,
        version: remoteEnd.version,
        proofTry: remoteEnd.proof(),
        proofHeight: localEnd.client.height,
      })
    else if (localEnd.state === OPEN && remoteEnd.state === TRYOPEN)
      // Handshake has confirmed locally (3 steps done), relay `chanOpenConfirm` to the remote end
      counterpartyDatagrams.push(ChanOpenConfirm{
        portIdentifier: remoteEnd.portIdentifier,
        channelIdentifier: remoteEnd.channelIdentifier,
        proofAck: localEnd.proof(),
        proofHeight: height
      })

    // Deal with packets
    // First, scan logs for sent packets and relay all of them
    sentPacketLogs = queryByTopic(height, "sendPacket")
    for (const logEntry of sentPacketLogs) {
      // relay packet with this sequence number
      packetData = Packet{logEntry.sequence, logEntry.timeoutHeight, logEntry.timeoutTimestamp,
                          localEnd.portIdentifier, localEnd.channelIdentifier,
                          remoteEnd.portIdentifier, remoteEnd.channelIdentifier, logEntry.data}
      counterpartyDatagrams.push(PacketRecv{
        packet: packetData,
        proof: packet.proof(),
        proofHeight: height,
      })
    }
    // Then, scan logs for received packets and relay acknowledgements
    recvPacketLogs = queryByTopic(height, "recvPacket")
    for (const logEntry of recvPacketLogs) {
      // relay packet acknowledgement with this sequence number
      packetData = Packet{logEntry.sequence, logEntry.timeoutHeight, logEntry.timeoutTimestamp,
                          localEnd.portIdentifier, localEnd.channelIdentifier,
                          remoteEnd.portIdentifier, remoteEnd.channelIdentifier, logEntry.data}
      counterpartyDatagrams.push(PacketAcknowledgement{
        packet: packetData,
        acknowledgement: logEntry.acknowledgement,
        proof: packet.proof(),
        proofHeight: height,
      })
    }
  }

  return [localDatagrams, counterpartyDatagrams]
}
```

中繼器可以選擇過濾這些數據報，以中繼特定的用戶端，特定的連接，特定的通道，甚至特定種類的封包，也許是根據費用支付模型（本文件未指定，因為它可能會各不相同）。

### 排序約束

在中繼器進程上存在隱式排序約束，以確定必須以什麼順序提交哪些數據報。例如，必須先提交區塊頭才能最終確定存儲在輕用戶端中特定高度的共識狀態和承諾根，然後才能轉發封包。兩條鏈直接的中繼器進程負責頻繁查詢兩條鏈的狀態，以確定何時必須中繼什麼。

### 捆綁

如果主機狀態機支持，則中繼器進程可以將許多數據報捆綁到一個交易中，這將導致它們按順序執行，並平攤所有開銷成本（例如，簽名檢查費用）。

### 競態條件

在同一對模組和鏈對之間進行中繼的多個中繼器可能會嘗試同時中繼同一封包（或提交相同的區塊頭）。如果有兩個中繼器這樣做，則第一個交易將成功，而第二個交易將失敗。為紓解這種情況，中繼器之間或發送原始封包的參與者與中繼器之間的帶外協調是必要的。進一步的討論超出了本標準的範圍。

### 激勵措施

中繼器進程必須有能夠訪問兩個鏈的具有足夠餘額的帳戶，以支付交易費用。中繼器可以在應用層採取別的方法來補償這些費用，例如透過在封包數據本身中包含少量費用——中繼器費用支付的協議將在此 ICS 的未來版本中或在單獨的 ICS 中進行描述。

可以安全的並行運行任意數量的中繼器進程（實際上，預計單獨的中繼器會服務於鏈間的單獨子集）。但是，如果他們多次提交相同的證明，則可能會花費不必要的費用，因此一些最小的協調可能是理想的（例如，將特定的中繼器分配給特定的封包或掃描記憶體池以查找未處理的交易）。

## 向後相容性

不適用。中繼器進程是鏈下的，可以根據需要進行升級或降級。

## 向前相容性

不適用。中繼器進程是鏈下的，可以根據需要進行升級或降級。

## 範例實現

即將到來。

## 其他實現

即將到來。

## 歷史

2019年3月30日-提交初稿

2019年4月15日-修訂格式和清晰度

2019年4月23日-注釋修訂;草案合併

## 版權

本文中的所有內容均根據 [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0) 獲得許可。
