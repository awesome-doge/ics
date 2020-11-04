---
ics: 1
title: ICS 規範標準
stage: 草案
category: 元標準
kind: 元標準
author: Christopher Goes <cwgoes@tendermint.com>
created: 2019-02-12
modified: 2019-08-25
---

## 什麼是 ICS？

鏈間標準（ICS）是一份描述會用於 Cosmos 生態系統的特定協議、標準或期望特性的設計文件。ICS 應該列出標準所需的屬性，解釋設計原理，並提供了一個簡明但全面的技術規範。ICS 的主要作者負責通過標準化流程推動提案，並徵求社區的投入和支持，並與利益相關者進行溝通，以確保（社會）共識。

鏈間標準化過程應是提出生態系統範圍協議，修改和特性的主要載體， 並且 ICS 文件應在達成共識後持久保留作為設計決策的紀錄和未來實施者的訊息存儲庫。

*不應*使用鏈間標準對特定區塊鏈進行修改（例如 Cosmos Hub），指定實現細節（例如特定程式語言的數據結構），或討論有關現有 Cosmos 區塊鏈的治理建議（儘管有可能，Cosmos 生態系統中的各個區塊鏈可以利用其治理流程批准或拒絕鏈間標準）。

## 組件

ICS 由標題，概要，規範，歷史記錄和版權聲明組成。所有頂級部分都是必需的。參考文獻應作為連結內聯，或在必要時以表格形式列在章節底部。

### 標題

ICS 標題包含與 ICS 相關的元數據。

#### 必填項

`ics: #` - ICS 編號（順序分配）

`題目` - ICS 題目（確保簡短）

`階段` - 當前 ICS 階段，請參見 [PROCESS.md](../../PROCESS.md) 獲取可能的階段列表。

有關 ICS 可接受階段的說明，請參見 [README.md](../../README.md)。

`類別` - ICS 類別，以下之一：

- `元標準` - 有關 ICS 流程的標準
- `IBC/TAO` - 有關區塊鏈間通信系統核心傳輸，身份認證和排序層協議的標準。
- `IBC/APP` - 關於區塊鏈間通信系統應用層協議的標準。

`作者` - ICS 作者和聯繫訊息（優先順序：電子郵件，GitHub，Twitter，其他可能得到回應的聯繫方法）。第一作者是 ICS 的主要“所有者”，並負責通過標準化過程進行推進。隨後的作者應按貢獻度排序。

`創建` - 首次創建 ICS 的日期（YYYY-MM-DD）

`修改` - ICS 上次修改日期（YYYY-MM-DD ）

#### 選填項

`依賴` - 此標準依賴的的其他 ICS 標準（使用編號引用）。

`依賴於` - 依賴此標準的其他 ICS 標準（使用編號引用）。

`替換` - 被此標準替代的另一個 ICS 標準， 如果適用。

`替換為` - 替代此標準的另一個 ICS 標準，如果適用。

### 概要

在標題之後，ICS 應該包含簡短的概要（約 200 個字），提供規範的高級描述和基本原理。

### 規範

規範部分是 ICS 的主要組成部分，應包含協議文件，設計原理，必需的參考以及適當的技術細節。

#### 子組件

規範可以包含任何或所有以下子組件，以適合特定的 ICS。包含的子組件應按此處指定的順序列出。

- *動機* - 提議特性或對現有特性的修改的根本依據。
- *定義* - 此 ICS 中使用的或理解該 ICS 所需的新術語或概念的列表。沒有在頂級“ docs”文件夾中定義術語必須在此處定義。
- *期望屬性* - 此協議所需的屬性或特性，以及違反這些屬性時的預期結果或錯誤的列表。
- *技術規範* - 所提議協議的所有技術細節，包括語法，語義，子協議，數據結構，算法和適當的虛擬碼。技術規範應足夠詳細，使獨立的正確實現彼此相容而不需要去了解其他規範。
- *向後相容性* - 討論與以前的功能或協議版本的相容性（或缺乏相容性）。
- *向前相容性* - 討論與未來可能或預期的功能或協議版本的相容性（或缺乏相容性）。
- *範例實現* - 具體的範例實現或對預期實現的描述，以作為實現者的主要參考。
- *其他實現* - 候選或最終實現的列表（外部引用，不要內聯）。

### 歷史

ICS 應該包括一個“歷史記錄”部分，列出所有啟發性的文件以及重大更改的純文本日誌。

請參見[下面](#history-1)的範例歷史記錄 。

### 版權

ICS 應該包含版權部分，按照 [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0) 的要求放棄權利。

## 格式化

### 通用

ICS 規範必須使用 GitHub 風格的 Markdown 編寫。

有關 GitHub 風格的 Markdown 速查表，請參見[此處](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet)。有關本地 Markdown 渲染器，請參見[此處](https://github.com/joeyespo/grip)。

### 語言

ICS 規範應使用簡單英語編寫，避免使用晦澀的術語和不必要的行話。有關簡單英語的絕佳範例，請參見 [Simple English Wikipedia](https://simple.wikipedia.org/wiki/Main_Page)。

規範中的關鍵字“必須”，“禁止”，“必選”，“應該”，“不應該”，“應當”，“不應當”，“建議”，“可”和“可選”按照 [RFC 2119](https://tools.ietf.org/html/rfc2119) 中的說明進行解釋。

### 虛擬碼

規範中的虛擬碼應與語言無關，並以簡單的命令式標準進行格式化，並帶有行號，變數，簡單的條件塊，for 循環和用於解釋進一步的功能的英語片段，例如計劃超時。應該避免使用 LaTeX 圖片，因為它們很難以 diff 形式進行查看。

用於結構體的虛擬碼應以簡單的 Typescript 作為介面編寫。

範例虛擬碼結構體：

```typescript
interface Connection {
  state: ConnectionState
  version: Version
  counterpartyIdentifier: Identifier
  consensusState: ConsensusState
}
```

用於算法的虛擬碼應以簡單的 Typescript 作為函數編寫。

範例虛擬碼算法：

```typescript
function startRound(round) {
  round_p = round
  step_p = PROPOSE
  if (proposer(h_p, round_p) === p) {
    if (validValue_p !== nil)
      proposal = validValue_p
    else
      proposal = getValue()
    broadcast( {PROPOSAL, h_p, round_p, proposal, validRound} )
  } else
    schedule(onTimeoutPropose(h_p, round_p), timeoutPropose(round_p))
}
```

## 歷史

該規範的靈感來自以太坊的 [EIP 1](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1.md)，其又依次源自比特幣的 BIP 流程和 Python 的 PEP 流程。以前的作者不對本 ICS 規範或 ICS 流程的不足負責。請將所有評論定向到 ICS 倉庫維護者。

2019年3月4日 - 初稿已完成並作為 PR 提交

2019年3月7日 - 草案合併

2019年4月11日 - 更新虛擬碼格式，添加定義小節

2019年8月17日 - 類別澄清

## 版權

本文中的所有內容均根據 [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0) 獲得許可。
