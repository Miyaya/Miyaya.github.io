---
title: "居然又開始刷 LeetCode - Binary search"
description: 
date: 2025-09-27T20:22:27+02:00
image:
math: 
license: 
hidden: false
comments: true
tags:
    - LeetCode
    - DSA
    - Golang
    - Anthropic Claude
    - Binary Search
categories:
    - dev
---

身為一個工程師，總覺得必須得時時更新自己的硬實力，沒寫點程式就覺得好像腦袋要長蜘蛛網了。 ~~也不排除是工作上的程式偏不好玩。~~ 總之，最近又開始刷題了。

不過比起以前還要一個個留言點開看解析，身在 AI 時代感覺刷題上有很明顯的差異!! 當然不是叫 AI 直接幫我做，而是我在 Claude 建立了一個 project，然後指定他為一個程式批改者，提供我以下的需求
- 指出目前程式的錯誤
- 說明主要想考什麼
- 提供 golang **最佳解** (因為我這次想練 go)
- 解釋以上程式碼的**時間跟空間複雜度**
- 提供 C++ 的解法 (因為我以往都寫 C++，感覺要面試也是 cpp 最普遍)
- 其他類似的題目

甚至我 project instruction 也都是用生成的! 

有一些以前始終想不通的盲點，最近一問之下醍醐灌頂。以前會覺得不斷刷題極其痛苦，對於普通人腦袋的我來說，刷題似乎只是在反覆驗證我很笨 இдஇ

現在有了個人家教的體驗，可以打破砂鍋問到底，刷題的體驗直接昇華成學習。但我依然覺得刷題是一條漫漫長路，這次沒有特別在找工作，先把筆記寫起來，未來有需要的話可以參考。

---

## 邊界定義遞迴範圍

二分搜尋不外乎最重要的是 boundary 設定，到底是 `[left, right]` 還是 `[left, right)` 首先要先確認。接著影響到迴圈或是遞迴的參數更新，這邊我都以迴圈為例，因為遞迴還會需要 extra stack memory。

```go
// [left, right) 
left, right := 0, len(nums)
for left < right {
    // mid = left + (right-left)/2
    // 更新：left = mid+1 或 right = mid
}

// [left, right]
left, right := 0, len(nums)-1
for left <= right {
    // mid = left + (right-left)/2  
    // 更新：left = mid+1 或 right = mid-1
}
```

> `[left, right]` 閉區間 → 使用 `left <= right`
使用閉區間時 left 和 right 都是有效的搜尋位置，當 `left == right` 時，還有一個位置沒檢查，所以條件必須是 `left <= right`。

```text
初始狀態：[1, 2, 3]
          ↑     ↑
        left  right
        
第一輪：mid = 2, 假設情境太小了，我們要取右半邊
       left = mid + 1 = 3
       
現在狀態：[_, _, 3]
                ↑
              left
              right
```
每次的檢查都是排除 mid（新的值為 mid - 1 或 mid + 1）當 `left == right == 3`，位置 3 還沒有檢查過！如果用 left < right，迴圈就結束了；如果用 left <= right，還會進行一次檢查 ✅

反之，在 `[left, right)` 的情況下，更新時右邊界不排除 mid，左邊界要排除，會變成mid + 1。

```go
left, right := 1, n
for left < right {
    mid := left + (right-left)/2
    switch guess(mid) {
    case 0:
        return mid
    case -1:
        right = mid      // 不排除 mid
    case 1:
        left = mid + 1   // 排除 mid
    }
}
```
### 錯誤範例

1. 閉區用 `<`

```go
// [left, right]
left, right := 1, n
for left < right {  // ❌ 會漏掉最後一個位置
    // ...
}
```

2. 半開區用 `<=`

```go
// [left, right)
left, right := 1, n
for left <= right {  // ❌ 會檢查無效位置 n
    // ...
}
```

3. 邊界條件不對
在需要排除 mid 的時候卻沒有排除，可能會造成無限迴圈

### 小結

```text
選擇邊界定義
├── 閉區間 [left, right]
│   ├── 初始化: left=1, right=n
│   ├── 條件: left <= right
│   └── 更新: right=mid-1, left=mid+1
│
└── 半開區間 [left, right)  
    ├── 初始化: left=1, right=n
    ├── 條件: left < right
    └── 更新: right=mid, left=mid+1
```

## 最後的 return value
> 關鍵觀察：
> - left 指向第一個不滿足條件的位置
> - right 指向最後一個滿足條件的位置
> 

以 **69. Sqrt(x)** 這題來說，假設 `x` 設定為 `8`，應該要回傳 `2`。我們來走一次演算法:

```text
初始狀態：left=1, right=4 (假設 right=x/2=4)
目標：找到最大的 k，使得 k² ≤ 8

第一輪：mid=2, 2²=4, 4 < 8 ✓
    答案可能是 2 或更大 → left = mid + 1 = 3
        
第二輪：left=3, right=4, mid=3, 3²=9, 9 > 8 ✗  
    答案不可能是 3 或更大 → right = mid - 1 = 2
        
第三輪：left=3, right=2, left > right
    迴圈結束
```

我們重新來審視一下終止條件：當 `left > right` 時迴圈結束，也就是說此時 right + 1 = left。

**在 [1, right] 範圍內，所有 k 滿足 k² ≤ x，在 [left, ∞) 範圍內，所有 k 滿足 k² > x。**

此時如果回傳 left 的話，由於 left 總是指向第一個不滿足條件的位置，即第一個使得 k² > x 的數字，這不是我們要的答案!

### 小結
尋找 **最後一個滿足條件的位置** → return right

尋找 **第一個滿足條件的位置** → return left