---
ics: 23
title: 向量承諾
stage: 草案
required-by: 2, 24
category: IBC/TAO
kind: 介面
author: Christopher Goes <cwgoes@tendermint.com>
created: 2019-04-16
modified: 2019-08-25
---

## 概要

*向量承諾*是一種構造，它對向量中任何索引和元素的成員資格/非成員資格的短證明產生恆定大小的綁定承諾。 本規範列舉了 IBC 協議中使用的承諾構造所需的函數和特性。特別是，IBC 中使用的承諾必須具有*位置約束力* ：它們必須能夠證明在特定位置（索引）的值存在或不存在。

### 動機

為了提供可以在另一條鏈上驗證的一條鏈上發生的特定狀態轉換的保證，IBC 需要一種有效的密碼構造來證明在狀態的特定路徑上包含或不包含特定值。

### 定義

向量承諾的*管理者*是具有在承諾中添加或刪除條目的能力和責任的參與者。通常，這將是區塊鏈的狀態機。

*證明者*是負責生成包含或不包含特定元素的證明的參與者。通常，這將是一個中繼器（請參閱 [ICS 18](../ics-018-relayer-algorithms) ）。

*驗證者*是檢查證明來驗證承諾的管理者是否添加了特定元素的參與者。通常，這將是在另一條鏈上運行的 IBC 處理程序（實現 IBC 的模組）。

使用特定的*路徑*和*值*類型實例化承諾，它們的類型假定為任意可序列化的數據。

一個*微不足道的函數*是增長速度比任何正多項式的倒數更慢的函數，如[這裡](https://en.wikipedia.org/wiki/Negligible_function)的定義 。

### 所需屬性

本文件僅定義所需的屬性，而不是具體的實現-請參見下面的“屬性”。

## 技術指標

### 數據類型

承諾構造必須指定以下數據類型，這些數據類型可以是不透明的（不需要外部檢視），但必須是可序列化的：

#### 承諾狀態

`CommitmentState`是承諾的完整狀態，將由管理器存儲。

```typescript
type CommitmentState = object
```

#### 承諾根

`CommitmentRoot`確保一個特定的承諾狀態，並且應為恆定大小。

在狀態大小恆定的某些承諾構造中， `CommitmentState`和`CommitmentRoot`可以是同一類型。

```typescript
type CommitmentRoot = object
```

#### 承諾路徑

`CommitmentPath`是用於驗證承諾證明的路徑，該路徑可以是任意結構化對象（由承諾類型定義）。它必須由通過`applyPrefix` （定義如下）計算出來。

```typescript
type CommitmentPath = object
```

#### 前綴

`CommitmentPrefix`定義承諾證明的存儲前綴。它在將路徑傳遞到證明驗證函數之前應用於路徑。

```typescript
type CommitmentPrefix = object
```

函數`applyPrefix`根據參數構造新的承諾路徑。它在前綴參數的上下文中解釋路徑參數。

對於兩個`(prefix, path)`元組， `applyPrefix(prefix, path)`必須僅在元組元素相等時才返回相同的鍵。

`applyPrefix`必須按`Path`來實現，因為`Path`可以具有不同的具體結構。 `applyPrefix`可以接受多種`CommitmentPrefix`類型。

`applyPrefix`返回的`CommitmentPath`並不需要是可序列化的（例如，它可能是樹節點標識符的列表），但它需要能夠比較是否相等。

```typescript
type applyPrefix = (prefix: CommitmentPrefix, path: Path) => CommitmentPath
```

#### 證明

一個`CommitmentProof`證明一個元素或一組元素的成員資格或非成員資格，可以與已知的承諾根一起驗證。證明應是簡潔的。

```typescript
type CommitmentProof = object
```

### 所需函數

承諾構造必須提供以下函數，這些函數在路徑上定義為可序列化的對象，在值上定義為位元組數組：

```typescript
type Path = string

type Value = []byte
```

#### 初始化

`generate`函數從一個路徑到值的映射（可能為空）初始化承諾的狀態。

```typescript
type generate = (initial: Map<Path, Value>) => CommitmentState
```

#### 根計算

`calculateRoot`函數計算承諾狀態的恆定大小的承諾，可用於驗證證明。

```typescript
type calculateRoot = (state: CommitmentState) => CommitmentRoot
```

#### 添加和刪除元素

`set`功能為承諾中的值設置路徑。

```typescript
type set = (state: CommitmentState, path: Path, value: Value) => CommitmentState
```

`remove`函數從承諾中刪除路徑和其關聯值。

```typescript
type remove = (state: CommitmentState, path: Path) => CommitmentState
```

#### 證明生成

`createMembershipProof`函數生成一個證明，證明特定承諾路徑已被設置為承諾中的特定值。

```typescript
type createMembershipProof = (state: CommitmentState, path: CommitmentPath, value: Value) => CommitmentProof
```

`createNonMembershipProof`函數生成一個證明，證明承諾路徑尚未設置為任何值。

```typescript
type createNonMembershipProof = (state: CommitmentState, path: CommitmentPath) => CommitmentProof
```

#### 證明驗證

`verifyMembership`函數驗證在承諾中已將路徑設置為特定值的證明。

```typescript
type verifyMembership = (root: CommitmentRoot, proof: CommitmentProof, path: CommitmentPath, value: Value) => boolean
```

`verifyNonMembership`函數驗證在承諾中尚未將路徑設置為任何值的證明。

```typescript
type verifyNonMembership = (root: CommitmentRoot, proof: CommitmentProof, path: CommitmentPath) => boolean
```

### 可選函數

承諾構造可以提供以下函數：

`batchVerifyMembership`函數驗證在承諾中已將多個路徑設置為特定值的證明。

```typescript
type batchVerifyMembership = (root: CommitmentRoot, proof: CommitmentProof, items: Map<CommitmentPath, Value>) => boolean
```

`batchVerifyNonMembership`函數可驗證證明在承諾中尚未將多個路徑設置為任何值的證明。

```typescript
type batchVerifyNonMembership = (root: CommitmentRoot, proof: CommitmentProof, paths: Set<CommitmentPath>) => boolean
```

如果定義這些函數，必須和使用`verifyMembership`和`verifyNonMembership`聯合在一起的結果相同（效率可能有所不同）：

```typescript
batchVerifyMembership(root, proof, items) ===
  all(items.map((item) => verifyMembership(root, proof, item.path, item.value)))
```

```typescript
batchVerifyNonMembership(root, proof, items) ===
  all(items.map((item) => verifyNonMembership(root, proof, item.path)))
```

如果批次驗證是可行的並且比單獨驗證每個元素的證明更有效，則承諾構造應定義批次驗證函數。

### 屬性和不變數

承諾必須是*完整的*，*合理的*和*有位置約束的*。這些屬性是相對於安全性參數`k`定義的，此安全性參數必須由管理者，證明者和驗證者達成一致（並且對於承諾算法通常是恆定的）。

#### 完整性

承諾證明必須是*完整的* ：已添加到承諾中的路徑/值映射始終可以被證明已包含在內，未包含的路徑始終可以被證明已被排除，除非是`k`定義的可以忽略的機率。

對於最後一個設置承諾`acc`中的值`value`的任何前綴`prefix`和任何路徑`path`，

```typescript
root = getRoot(acc)
proof = createMembershipProof(acc, applyPrefix(prefix, path), value)
```

```
Probability(verifyMembership(root, proof, applyPrefix(prefix, path), value) === false) negligible in k
```

對於沒有在承諾`acc`中設置的任何前綴`prefix`和任何路徑`path` ，對於`proof`的所有值和`value`的所有值的

```typescript
root = getRoot(acc)
proof = createNonMembershipProof(acc, applyPrefix(prefix, path))
```

```
Probability(verifyNonMembership(root, proof, applyPrefix(prefix, path)) === false) negligible in k
```

#### 合理性

承諾證明必須是*合理的* ：除非在可配置安全性參數`k`機率下可以忽略不計，否則不能將未添加到承諾中的路徑/值映射證明為已包含，或者將已經添加到承諾中的路徑證明為已排除。

對於最後一個設置值`value`在承諾`acc`中的任何前綴`prefix`和任何路徑`path`，對於`proof` 的所有值，

```
Probability(verifyNonMembership(root, proof, applyPrefix(prefix, path)) === true) negligible in k
```

對於沒有在承諾`acc`中設置的任何前綴`prefix`和任何路徑`path`，對於`proof`的所有值和`value`的所有值 ，

```
Probability(verifyMembership(root, proof, applyPrefix(prefix, path), value) === true) negligible in k
```

#### 位置綁定

承諾證明必須是*有位置約束的* ：給定的承諾路徑只能映射到一個值，並且承諾證明不能證明同一路徑適用於不同的值，除非在機率k下可以被忽略。

對於在承諾`acc`設置的任何前綴`prefix`和任何路徑`path` ，都有一個`value` ：

```typescript
root = getRoot(acc)
proof = createMembershipProof(acc, applyPrefix(prefix, path), value)
```

```
Probability(verifyMembership(root, proof, applyPrefix(prefix, path), value) === false) negligible in k
```

對於所有其他值`otherValue` ，其中`value !== otherValue` ，對於`proof`的所有值，

```
Probability(verifyMembership(root, proof, applyPrefix(prefix, path), otherValue) === true) negligible in k
```

## 向後相容性

不適用。

## 向前相容性

承諾算法將是固定的。可以透過對連接和通道進行版本控制來引入新算法。

## 範例實現

即將到來。

## 其他實現

即將到來。

## 歷史

安全性定義主要來自這些文章（並進行了一些簡化）：

- [向量承諾及其應用](https://eprint.iacr.org/2011/495.pdf)
- [應用程式對保留匿名撤銷的承諾](https://eprint.iacr.org/2017/043.pdf)
- [用於 IOP 和無狀態區塊鏈的承諾批處理技術](https://eprint.iacr.org/2018/1188.pdf)

感謝 Dev Ojha 對這個規範的廣泛評論。

2019年4月25日-提交的草稿

## 版權

本文中的所有內容均根據 [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0) 獲得許可。
