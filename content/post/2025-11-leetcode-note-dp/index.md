---
title: "DP 與狀態轉移"
description: 
date: 2025-11-12T17:28:42+01:00
image:
math: 
license: 
hidden: false
comments: true
draft: false
tags:
    - LeetCode
    - DSA
    - Golang
    - Dynamic Programming
categories:
    - dev
---

因為 DP 是面試常遇到的問題，此文會從 DP 演算法的演化開始解釋，並說明如何優化時間與空間複雜度，最後以如何識別題型、尋找 DP 關注點、程式模板結尾。

## 當 backtracking TLE 
DP 應用的場景很多。一開始的概念是由 backtrack 方法而生，由於 backtrack 會將所有可能性遍歷，其中的很多 path 會被重複計算，為了節省效率、不重複計算，我們可以將部分計算結果記起來，當需要參照時，只需要直接查表即可。

讓我們首先回顧 backtrack 的概念

```golang
backtrack(參數) {
    if 終止條件 { 保存結果; return }
    for 選擇 in 選擇列表 {
        做選擇
        backtrack(新參數)
        撤銷選擇
    }
}
```

### [746. Min Cost Climbing Stairs](https://leetcode.com/problems/min-cost-climbing-stairs/)

> You are given an integer array cost where cost[i] is the cost of ith step on a staircase. Once you pay the cost, you can either climb one or two steps.
> 
> You can either start from the step with index 0, or the step with index 1.
>
> Return the minimum cost to reach the top of the floor.
> 
> Example 1:
> 
> Input: cost = [10,15,20]
> Output: 15
>
> Explanation: You will start at index 1.
> Pay 15 and climb two steps to reach the top.
> The total cost is 15.

以這題為例，若使用 backtracking 的話就是每一次都去回溯前一步與前兩步，然後取其小值。

```golang
func minCostClimbingStairs(cost []int) int {
    n := len(cost)
    
    var backtrack func(i int) int
    backtrack = func(i int) int {
        if i >= n { // base condition
            return 0
        }
        
        oneStep := cost[i] + backtrack(i+1)
        twoSteps := cost[i] + backtrack(i+2)
        
        return min(oneStep, twoSteps)
    }
    
    return min(backtrack(0), backtrack(1))
}
```

**空間複雜度:** O(n)

**時間複雜度:** O(2<sup>n</sup>)
因為每個位置都有兩個選擇：爬一階或兩階，因此會長成一棵高度為 n 的二元決策樹，總共的 path 在每一個節點都是兩者擇一，因此來到了 2<sup>n</sup> 的時間複雜度。

視覺化的話可以看到中間的階梯「到2」、「到3」、「到4」都被重複計算了很多次!

```
cost = [10, 15, 20, 25]

                    起點
                  /      \
            從0開始(10)   從1開始(15)
              /    \        /    \
          到1(15)  到2(20)  到2(20)  到3(25)
           /  \     /  \     /  \     /  \
        到2  到3  到3  到4  到3  到4  到4  到5
       (20) (25) (25)     (25)
```

## Memoization (top-down)

承上所說，我們想把重複的部分紀錄起來，因此宣告一個 array 名為 memo，`memo[i]` 紀錄到達第 i 階的最小步數。如果 `memo[i]` 沒有資料才需要計算，並且在計算完後把 data 存起來！

```go
func minCostClimbingStairs(cost []int) int {
    n := len(cost)
    memo := make([]int, n)
    for i := range memo {
        memo[i] = -1
    }
    
    var dp func(i int) int
    dp = func(i int) int {
        if i >= n {
            return 0
        }
        
        // 如果已經計算過,直接返回記錄的結果
        if memo[i] != -1 {
            return memo[i]
        }
        
        // 計算並記錄結果
        oneStep := cost[i] + dp(i+1)
        twoSteps := cost[i] + dp(i+2)
        memo[i] = min(oneStep, twoSteps)
        
        return memo[i]
    }
    
    return min(dp(0), dp(1))
}
```

### 複雜度分析

**時間複雜度:** O(n)
共會計算 n 次，因為重複的部分透過 O(1) 的時間複雜度直接查。

**空間複雜度:** O(n)
使用到額外的 memo array O(n) 與 stack 大小 O(n)，O(2n)=O(n)

在此可以注意到，如果我們想要降低空間複雜度，首當其要是把遞迴改成迴圈，誠如 backtracking 遇到的情況。

## Bottom-Up DP
核心思路為**從最簡單的子問題開始，逐步構建到完整問題的解**。以同樣的一題，定義 dp[i] 為到達*第 i 階的最小成本*。

```go
func minCostClimbingStairs(cost []int) int {
    n := len(cost)
    dp := make([]int, n+1)
    
    // base condition
    dp[0] = 0
    dp[1] = 0
    
    for i := 2; i <= n; i++ {
        dp[i] = min(dp[i-1]+cost[i-1], dp[i-2]+cost[i-2])
    }
    
    return dp[n]
}
```

### 狀態轉移方程
```go
dp[i] = min(dp[i-1] + cost[i-1], dp[i-2] + cost[i-2])
```

在這邊有一句很重要的程式，決定了 dp 表格是如何一步步建構起來。我們命名為**狀態轉移方程**。

#### 什麼是「狀態」

狀態就是在解決問題的某個時刻，我們需要知道的所有關鍵資訊。想像你在玩一個遊戲，狀態就像是遊戲的「存檔點」：
- 你在哪裡？（位置）
- 你有什麼？（資源）
- 你做過什麼？（歷史）

以 Min Cost Climbing Stairs 這題為例：
問題：*要爬到樓梯頂部*

可能的狀態定義方式：

**方式 1** -> 上面的程式採用
> 狀態 = "我現在站在第 i 階"
>
> 記錄 = "到達第 i 階的最小成本"

**方式 2** -> 也可以
> 狀態 = "我要從第 i 階出發"
> 
> 記錄 = "從第 i 階到頂部的最小成本"

**方式 3** -> 太複雜！不需要記錄這麼多
> 狀態 = "我在第 i 階，而且我是從第 j 階來的"
> 
> 記錄 = "到達第 i 階並且上一步在第 j 階的最小成本"

#### 什麼是「狀態轉移」
狀態轉移就是從一個狀態到另一個狀態的變化規則。就像遊戲中的「行動規則」：

我在 A 點，可以做什麼動作到達 B 點？
這個動作的代價是什麼？

通用形式：
```go
dp[當前狀態] = function(dp[之前的狀態們], 當前決策)
```

逐項解釋的話，這裡的狀態是這樣轉移的
```
dp[i] = min(dp[i-1] + cost[i-1], dp[i-2] + cost[i-2])
        ↑   ↑                    ↑
        |   |                    |
     函數  從i-1來的路徑        從i-2來的路徑
```

### 決策流程圖
```
開始
  ↓
問題：我要求什麼？
（最大值、最小值、方案數、是否可行...）
  ↓
確定目標狀態
（通常是最後一步、終點、目標數值）
  ↓
反推：達到目標需要什麼資訊？
  ↓
這些資訊會隨著什麼改變？
（位置、時間、剩餘資源...）
  ↓
能否用簡單的變數表達這些變化？
  ├─ 是 → 這就是你的狀態定義
  └─ 否 → 需要更多維度或重新思考
```

再次使用這題為例：

1. 我要求什麼？

答：到達樓梯頂部的最小成本

2. 目標狀態－樓梯頂部是什麼？

答：第 n 階（cost 陣列長度之後）

3. 到達第 n 階需要什麼資訊？

答：我需要知道到達前面各階的成本

4. 這些資訊隨什麼改變？

答：隨著樓梯的階數 i 改變

5. 如何表達？

答：dp[i] = 到達第 i 階的最小成本

---

如此一來，就可以找到正確的狀態轉移程式！

## 空間優化
終於來到最後一步！觀察狀態轉移方程，發現 dp[i] 只依賴於 dp[i-1] 和 dp[i-2]，所以不需要保存整個陣列，只需要保存最近的兩個值。

```go
func minCostClimbingStairs(cost []int) int {
    n := len(cost)
    
    // 只需要保存兩個狀態
    // prev2: 代表 dp[i-2]
    // prev1: 代表 dp[i-1]
    prev2 := 0  // dp[0]
    prev1 := 0  // dp[1]
    
    for i := 2; i <= n; i++ {
        current := min(prev1+cost[i-1], prev2+cost[i-2])
        
        // 滑動窗口:更新兩個變數
        prev2 = prev1
        prev1 = current
    }
    
    // prev1 現在儲存的是 dp[n]
    return prev1
}
```

### 複雜度分析

**時間複雜度:** O(n)

**空間複雜度:** O(1)

只使用了固定數量的變數(prev2, prev1, current)，不隨輸入大小增長

## 結論 —— DP 通用解題框架
> 遞迴 → 遞迴+記憶化 → DP → 空間優化 DP

**步驟 1: 識別 DP 問題特徵**
- ✓ 有**最優子結構**(大問題的最優解包含子問題的最優解)
- ✓ 有**重疊子問題**(相同的子問題被多次求解)
- ✓ 需要求**最優解**(最大值、最小值、最長、最短等)

**步驟 2: 定義狀態**: "我需要記錄什麼資訊來解決問題?"

**步驟 3: 找出狀態轉移方程**: "當前狀態如何從之前的狀態推導?"

**步驟 4: 確定基礎情況**: "最簡單的情況是什麼?"

**步驟 5: 決定計算順序**
- Top-Down: 從目標開始,遞迴到基礎情況 (需要 memoization)
- Bottom-Up: 從基礎情況開始,迭代到目標 (通常更高效)

**步驟 6: 空間優化 (如果可能)**
