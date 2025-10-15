---
title: "LeetCode250 進度10% - Heap"
description: 
date: 2025-10-01T18:19:32+02:00
image:
math: 
license: 
hidden: true
comments: false
tags:
    - LeetCode
    - DSA
    - Golang
    - Heap
    - Priority Queue
categories:
    - dev
---

## 題目類型
今天要來寫 ***Top K ...*** 這類的題型，通常只要一看到就可以先聯想到 Heap，因為 heap 的複雜度會隨著 `k` 而改變。舉例來說，使用 heap 排序的話，時間複雜度為 O(nlogk)、空間複雜度為 O(n+k)。

- 692. Top K Frequent Words - 字串版本，需要考慮字典序
- 215. Kth Largest Element in an Array - 找第 k 大元素
- 451. Sort Characters By Frequency - 按頻率排序字元
- 295. Find Median from Data Stream - 動態中位數
- 373. Find K Pairs with Smallest Sums - K 個最小和的配對
- 378. Kth Smallest Element in a Sorted Matrix - 排序矩陣中第 k 小

## 實作
C++ 可以直接用 `priority_queue` 來進行 heap 的實作，但 Go 有其限制，因此要先定義一些方法的實作，然後才可以呼叫 `heap`:

```go
import "container/heap"

// 實作 heap.Interface
type IntHeap []int
```

1. **`Len`**
這邊 golang 使用 slice 實作 heap 的底層資料結構，`Len` 回傳此 slice 的大小

```go
func (h IntHeap) Len() int           { return len(h) }
```

2. **`Less`**
定義此 heap 是如何排序，舉例來說 max heap 的 `Less` 比較方式會由小於改為大於。

```go
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] } // min heap
```

如果 slice type 改變，如

```go
type Element struct {
    num   int
    freq  int
    index int
}
```

則可以定義排序時要比較 num, freq, index 哪幾個、怎麼比較。

3. **`Swap`**
定義當兩個 item 交換時的原則，平常寫程式時不會用到，主要應該是 `Pop` 與 `Push` 的底層機制

```go
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
```

4. **`Push`**
注意這邊的 receiver 是要傳入*指標*，因為當新增元素，可能會改變 slice 的大小 (len)，因此若沒有用指標將會 pass by value，那麼實際的 slice 並沒有被更動到。

```go
func (h *IntHeap) Push(x interface{}) {
    *h = append(*h, x.(int))
}
```

而且，這邊的 `Push` 僅定義如何增加，看到這邊，可以重新回想一下 Heap 是如何新增物件的，通常我們會 append 到 array 尾端，然後再一層層根據比較依據往上浮。我們只需負責前半段—— append 到 slice 最後一個元素，然後剩下的 heap 套件將會幫我們完成演算法。

```go
// heap 套件的核心函數
func Push(h Interface, x interface{}) {
    h.Push(x)           // 你的 Push: 將元素加到底層切片
    up(h, h.Len()-1)   // heap 套件的 up: 向上調整維持 heap 性質
}

func Pop(h Interface) interface{} {
    n := h.Len() - 1
    h.Swap(0, n)       // 交換第一個和最後一個
    down(h, 0, n)      // 向下調整
    return h.Pop()     // 你的 Pop: 從底層切片移除元素
}
```

5. **`Pop`**
同樣的，`Pop` 我們僅需定義 slice 實作——移除最後一個元素，並回傳原本的第一個元素。交換的過程以及重新使得 heap 遵守 heap 元素之前的關係則是套件會幫我們完成。

```go
func (h *IntHeap) Pop() interface{} {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[0 : n-1]
    return x
}
```

6. **`heap.Init`**
當定義好的 slice 提供充分 heap.Interface 實作，就可以呼叫 `heap.Init(slice)`。

```go
func main() {
    // 方式 1: 直接建立
    h := &IntHeap{3, 1, 4, 1, 5}
    heap.Init(h)
    fmt.Println(*h)  // [1 1 4 3 5]
    
    // 方式 2: 從現有 slice 轉換
    nums := []int{3, 1, 4, 1, 5}
    h2 := (*IntHeap)(&nums)  // 類型轉換
    heap.Init(h2)
    fmt.Println(*h2)  // [1 1 4 3 5]
}
```

### 分工
| 你實作的方法 | 負責什麼 | heap 套件負責什麼 |
| ---------- | ------- | --------------- |
| Len()      | 告訴長度 |決定何時需要知道長度 |
| Less()     | 定義比較規則 | 決定何時需要比較 |
| Swap()     | 交換兩個元素 | 決定交換哪兩個 |
| Push()     | 把元素加到末尾 | 決定加入後如何調整 |
| Pop()      | 從末尾移除元素 | 決定移除前如何調整 |


## 範例
以 **347. Top K Frequent Elements** 為例，我們使用剛剛的練習來進行實作

```go
func topKFrequent(nums []int, k int) []int {
    // 步驟 1: 統計每個元素的頻率
    freqMap := make(map[int]int)
    for _, num := range nums {
        freqMap[num]++
    }
    
    // 步驟 2: 使用 min heap 維護前 k 個最高頻率元素
    h := &MinHeap{}
    heap.Init(h)
    
    for val, freq := range freqMap {
        if h.Len() < k {
            // heap 尚未滿，直接加入
            heap.Push(h, Element{val: val, freq: freq})
        } else if freq > (*h)[0].freq {
            // 當前頻率比 heap 頂部大，替換
            heap.Pop(h)
            heap.Push(h, Element{val: val, freq: freq})
        }
    }
    
    // 步驟 3: 從 heap 中提取結果
    result := make([]int, k)
    for i := k - 1; i >= 0; i-- {
        result[i] = heap.Pop(h).(Element).val
    }
    
    return result
}
```

### 與 C++ 比較
C++ 可以直接使用 STL priority_queue，使用上來更方便，雖然 golang 的介面定義提供了更明確的靈活性。另外，C++ STL 預設是 max heap。

```cpp
#include <queue>
#include <unordered_map>

// 自動管理的 max heap
priority_queue<pair<int, int>> pq;

// 或者使用 min heap
priority_queue<pair<int, int>, vector<pair<int, int>>, greater<pair<int, int>>> minPq;
```

## 剩下的 90% 漫漫長路
以前刷題會因為難度與時間壓力而感到恐懼，進而對面試感到沒有信心，至今這件事仍沒有改變。我覺得我像是一個科目沒有學好的學生，學完了一次，考差了。再學一次，還是學得很差。反覆的過程很像一直在原地考試，明明都是一樣的內容但考的終究很差。

但我好像逐漸接受刷題是一個漫長的過程，至少對我來說我沒有辦法像那些聰明的人類狂 cram 三個月然後可以直接 20 分鐘幹掉多數 medium/hard questions。我想到以前大學有個同學，不能否認我所有的同學都聰明的要死，但有一個同學兩個學期的微積分都被當掉了，她完全不是像我這種臨時抱佛腳或是擺爛的類型，但有可能她就是某個地方卡住了，然後她一點點克服，後來還是考上研究所。

我應該也要學習像那樣。希望有朝一日我能變得很強。

更別說還有一回目、二回目、三回目的反覆練習。最終的目標是能即時反應題目，並注意到 edge case 的細節且能夠試想到。