---
title: "圖論 101"
description: 圖的定義、環的檢驗、相鄰點的表示法
date: 2025-12-03T21:51:43+01:00
image:
math: true
license: 
hidden: false
comments: true
draft: false
tags:
    - LeetCode
    - DSA
    - Golang
    - Graphs
    - NeetCode
categories:
    - leetcode
---

刷題來到了圖論⋯⋯發現忘了很多基本的觀念。

因此這篇就來概念性的複習，像是圖的定義、經典的 DAG 與拓樸排序、如何表示節點之間相鄰關係。有機會的話會再根據常見題型細談演算法的部分，像是找最短路徑、遍歷、最小生成樹等。

## 定義
圖由頂點 (Vertex) 以及邊 (Edge) 構成，常用來表示事物之間的關係，如狀態的順序、社交的關係、地點的相連。

### 有向圖 vs. 無向圖
無向圖僅表示兩個頂點之間有連結，帶有對稱性。也就是說，當 A 點會到 B 點，那麼反之亦然則 B→A 也存在。

有向圖則是「單向」：表示了兩個頂點從一點到另一點是存在的。不過，A→B 存在不代表 B→A 一定成立。

### 入度 vs. 出度
入度 (in-degree) 代表指向該頂點的邊數，出度 (out-degree) 則相反。但在無向圖並不會提是入或出 (畢竟是一樣的東西)。

入度跟出度可以用來形容這個圖的情況，舉例來說訂閱或關注的關係、任務的相依賴性。

---

此外，也可以用這個作為一些特別 case 的條件，如我們可以說 **tree 是一種特例的圖**: 
1. 若有 n 個節點，則應該存在 n-1 個邊
2. 全部的點都要有辦法相連、不能有孤立到達不了的頂點 (每個點最少的度數為 1)
3. 不能有環

#### 握手定理 Handshaking Lemma
因為每條邊貢獻一入一出 2 個度，所以所有節點的度之和 = 2 × 邊數
  $$
  \sum_{v \in V} \text{deg}(v) = 2|E|
  $$

---

在拓樸排序中，入度將是一個很好的指示: 當 `in-degree == 0` 時，代表沒有依賴於這個頂點的其他點，因此可以執行。

#### 拓樸排序 Kahn's algorithm
```go
func topologicalSort(n int, edges [][]int) []int {
    adjList := make([][]int, n)
    indegree := make([]int, n)
    
    for _, edge := range edges {
        adjList[edge[0]] = append(adjList[edge[0]], edge[1])
        indegree[edge[1]]++
    }
    
    queue := make([]int, 0)
    for i := 0; i < n; i++ {
        if indegree[i] == 0 {
            queue = append(queue, i)
        }
    }
    
    result := make([]int, 0)
    front := 0
    
    for front < len(queue) {
        node := queue[front]
        front++
        result = append(result, node)
        
        for _, neighbor := range adjList[node] {
            indegree[neighbor]--
            if indegree[neighbor] == 0 {
                queue = append(queue, neighbor)
            }
        }
    }
    
    if len(result) != n {
        return nil  // 有環
    }
    
    return result
}
```

## 鄰居的表示方法

### 鄰接矩陣
`matrix[i][j]` 表示 i 到 j 的邊
```go
// 無向圖
matrix := make([][]int, n)
for i := range matrix {
    matrix[i] = make([]int, n)
}

// 添加邊 u-v
matrix[u][v] = 1
matrix[v][u] = 1  // 無向圖對稱

// 加權圖
matrix[u][v] = weight
matrix[v][u] = weight
```

### 鄰接表
```go
// 無向圖
adjList := make([][]int, n)
adjList[u] = append(adjList[u], v)
adjList[v] = append(adjList[v], u)

// 有向圖
adjList[u] = append(adjList[u], v)  // u → v

// 加權圖
type Edge struct {
    To     int
    Weight int
}
adjList := make([][]Edge, n)
adjList[u] = append(adjList[u], Edge{v, weight})
```

### 邊列表
```go
type Edge struct {
    From   int
    To     int
    Weight int  // 可選
}

edges := []Edge{
    {0, 1, 2},
    {1, 2, 3},
    {2, 3, 1},
}
```

## 度的計算複雜度
鄰接表跟鄰接矩陣都是紀錄每一點與其他點的相鄰狀態，數學上的語意即「從該點出發到別的點的 E 是否存在」，因此當計算出度時只要看這個點上的鄰接表大小。

而鄰接矩陣則是把每一個點對點的關係建立出來所以佔用了 V² 的空間，當我們想要透過鄰接矩陣計算出度或入度時需要遍歷整欄或整列以確認對於其他點的連接狀態。

| 操作 | 鄰接表 adj list | 鄰接矩陣 adj matrix |
|------|--------|----------|
| **計算出度** | O(1) | O(V) |
| **計算入度** | O(E) 預處理 | O(V) |
| **所有節點的度** | O(V+E) | O(V²) |

### 遍歷
因為鄰接表的特性，因此在特定題目需要遍歷的情況，我們可以將時間複雜度降低到 O(V+E)。

如 [207. Course Schedule](https://leetcode.com/problems/course-schedule/) 這一題，雖然有兩層循環，但每條邊只被訪問一次，每個節點只被處理一次。

假設有 6 個節點、5 個邊，外層會跑 6 次，但**內層循環的迭代次數不是固定的 V，而是取決於當前節點的出度**，也就是說在這個例子中，裡面的回圈因為只執行每個節點的鄰居，所以兩層內圈的執行次數是 5 = E。**此處我們並不會說是 O(VE)，而是 *O(V+E)*。**
```go
for i := 0; i < numCourses; i++ { // 執行 V 次
    courseIndex := queue[0]
    queue = queue[1:]
    
    for _, course := range unlock[courseIndex] {  // 只處理鄰居
        inDegree[course]--
        if inDegree[course] == 0 {
            queue = append(queue, course)
        }
    }
}
```

我們再使用一些極端情況來驗證

#### **情況 1: 稀疏圖 (E ≈ V)**
圖: 0→1→2→3→4 (鏈狀)

E = 4, V = 5

```
unlock[0] = [1]     執行 1 次
unlock[1] = [2]     執行 1 次
unlock[2] = [3]     執行 1 次
unlock[3] = [4]     執行 1 次
unlock[4] = []      執行 0 次
```

總執行次數 = 4 = E

時間複雜度 = O(V + E) = O(V) (因為 E ≈ V)

#### **情況 2: 稠密圖 (E ≈ V²)**
圖: 完全圖,每個節點連到其他所有節點

V = 4, E = 12 (每個節點有 3 條出邊)

```
unlock[0] = [1,2,3]    執行 3 次
unlock[1] = [0,2,3]    執行 3 次
unlock[2] = [0,1,3]    執行 3 次
unlock[3] = [0,1,2]    執行 3 次
```

總執行次數 = 12 = E

時間複雜度 = O(V + E) = O(E) (因為 E >> V)

## 環與路徑
路徑 (path) 在圖中代表一連串有順序的節點，這些節點先後有連接起來。

環 (cycle) 的定義則是起點與終點為同一個點的路徑，且路徑有一條邊，也就是說可以是一個邊指向自己。在有向圖與無向圖中都可能有環的出現。

### 檢驗環
我自己覺得有向環比較簡單，因為我們可以遍歷邊，並將拜訪過的節點標注起來，在遍歷過程中，如果遇到已經拜訪過的點，就代表環存在。
或是也可以使用前面提到的拓樸排序，因為拓樸排序限定於有向無環圖 (DAG)，所以如果沒有辦法做拓樸排序，就代表其中有環！

```go
func hasCycle(n int, edges [][]int) bool {
    adjList := make([][]int, n)
    indegree := make([]int, n)
    
    for _, edge := range edges {
        adjList[edge[0]] = append(adjList[edge[0]], edge[1])
        indegree[edge[1]]++
    }
    
    queue := make([]int, 0)
    for i := 0; i < n; i++ {
        if indegree[i] == 0 {
            queue = append(queue, i)
        }
    }
    
    count := 0
    front := 0
    
    for front < len(queue) {
        node := queue[front]
        front++
        count++
        
        for _, neighbor := range adjList[node] {
            indegree[neighbor]--
            if indegree[neighbor] == 0 {
                queue = append(queue, neighbor)
            }
        }
    }
    
    // 如果處理的節點數 < 總節點數 → 有環
    return count != n
}
```

無向圖的部分，因為在遍歷的過程中沒有辦法確定目前訪問的點如果已經訪問過是否代表有環，因此需要特別標注這個點是不是父節點，是的話就可以正常回溯、不是的話才代表有環。
```go
func hasCycle(n int, edges [][]int) bool {
    adjList := make([][]int, n)
    for _, edge := range edges {
        adjList[edge[0]] = append(adjList[edge[0]], edge[1])
        adjList[edge[1]] = append(adjList[edge[1]], edge[0])
    }
    
    visited := make([]bool, n)
    
    var dfs func(node, parent int) bool
    dfs = func(node, parent int) bool {
        visited[node] = true
        
        for _, neighbor := range adjList[node] {
            if !visited[neighbor] {
                if dfs(neighbor, node) {
                    return true
                }
            } else if neighbor != parent {
                // 訪問到已訪問節點,且不是父節點 → 環
                return true
            }
        }
        return false
    }
    
    // 檢查所有連通分量
    for i := 0; i < n; i++ {
        if !visited[i] {
            if dfs(i, -1) {
                return true
            }
        }
    }
    
    return false
}
```


## Black friday & cyber monday
買了很多東西，包含送家人的聖誕禮物、辦公用品、還訂了一年份 LeetCode Premium，發現也不貴，當初刷題 [LeetCode 75](https://leetcode.com/studyplan/leetcode-75/) 後拿到了兩個很不錯的實習機會，接著刷 [Top Interview 150](https://leetcode.com/studyplan/top-interview-150/) 也順利地找到 CDI，希望可以透過多練習，真的精通 DSA 然後去規模更大的公司。