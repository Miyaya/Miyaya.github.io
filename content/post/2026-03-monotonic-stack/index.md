---
title: "NeetCode250 進度 68% - Monotonic Stack"
description: 2026 第一篇文章✨
date: 2026-03-07T16:56:36+01:00
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
    - Stack
    - Monotonic Stack
    - NeetCode
categories:
    - leetcode
---

## 2026 開局
年初去滑雪時跌倒，導致這兩個多月幾乎足不出戶，不知是禍是福 XD 總之還是有加減繼續刷題，雖然進入後半段開始有比較難的 medium 跟很多根本沒有辦法在半小時內想出來的 hard，導致厭世值++ 然後刷題速度變得很慢，還是持續的努力。

希望可以在這個 12week-year 把 NeetCode250 的目標達成。

### 農曆年看甄嬛馬拉松
平常我是很討厭過農曆年的，小時候很羨慕大家庭那種熱鬧哄哄的感覺，但我自己的經驗告訴我，我們家逐年是越加分崩離析。因此離開台灣是躲避過年的完美藉口，看到別人說很想念跟家人吃年菜、打麻將什麼的，我也沒有這種美好回憶，自然一點都不想念。反而因為去年看了甄嬛傳，今年終於有機會追甄嬛馬拉松，不得不說這是我過過最溫馨、熱鬧的農曆年了。

雖然被我描述得有點悲慘，但我是很滿意且感恩的。

## Stack
言歸正傳，今天想聊 Monotonic stack，就不得不先提到 Stack。

大家都知道 stack 是 FILO (first-in-last-out) 特性的資料結構，c++ 的實作可以使用標準庫 STL 定義的 stack。但 go 沒有，只能用 `[]int` 來實作，倒也不難。

時間複雜度在 push 和 pop 都是 O(1)。

### Monotonic stack
在 stack 的基礎上， monotonic stack 增加了單調遞增 (monotone increasing) 或單調遞減 (monotone decreasing) 的限制。

也就是說，stack 從底部一路往上只能遞增，top 會是最大值，或是一路只能遞減，top 會是最小值。當要 push 進新的元素時，若違反了這個單調性，就會不斷地 pop 直到滿足條件為止。

把這個邏輯寫成 pseudo code 的話如下

```
for 每個新元素 x:
    while 棧不為空 AND 棧頂元素 >= x:
        彈出棧頂元素（此時可以處理被彈出的元素）
    將 x 入棧
```

#### 複雜度
時間複雜度，因為每個元素最多 push/pop 各一次，所以複雜度為 O(n)。

空間複雜度的話，考慮整串入 stack O(n)。

## 適用場景
- 下一個更大 / 更小的元素（Next Greater / Smaller Element）
- 前一個更大 / 更小的元素（Previous Greater / Smaller Element）
- 在某範圍內找最大/最小（柱狀圖、溫度、股價）
- 計算跨度（Span）/ 距離

### 範例與模板

#### 739. Daily Temperatures
題目： 給一個氣溫陣列 temperatures，對每天求出「幾天後會遇到更高溫度」，若之後沒有更高溫則為 0。

我們可以使用一個單調遞減的 stack 存 index，當當前溫度 > stack.top() 的溫度時，說明找到了「更暖的那天」，把 index pop 出來並計算天數差就可以得到「幾天後遇到更高溫」的結果。

**實作**

注意前面提到 golang 沒有 stack 這個資料結構，因此 top() 的等值是 array 裡的最後一個元素。
pop() 的實作則需要手動提取最後一個元素，然後再重新 assign array。
```go
func dailyTemperatures(temperatures []int) []int {
    n := len(temperatures)
    result := make([]int, n)
    stack := []int{} // index

    for i := 0; i < n; i++ {
        
        for len(stack) > 0 && temperatures[i] > temperatures[stack[len(stack)-1]] {
            // pop
            idx := stack[len(stack)-1]
            stack = stack[:len(stack)-1]

            result[idx] = i - idx
        }
        stack = append(stack, i)
    }
    return result
}
```

因為題目說預設找不到的話回傳 0 即可，所以不用特別額外處理還留在 stack 的 index。

#### 84. Largest Rectangle in Histogram
題目： 柱狀圖中找面積最大的矩形。

如果要求面積的話，針對每個 height，我們想要知道左右的邊界（總共的寬是多少）然後乘以 height 本身。而邊界的定義為當鄰居的值小於 height 本身（因為就不再是矩形了）。

講到這裡，先設計一個資料結構存左邊界與 height，並使用單調遞增 stack，當遇到比 stack.top() 更矮的 height 時，代表 top.height 沒有辦法往右延伸，因此可以 pop 出來計算面積。每次計算面積的時候也要更新左邊界，因為 pop 出來的時候代表當前還沒 push 進去的 height 是可以往左邊延伸的（因為比較矮）。

**實作**
```go
type Pair struct {
    index int // left boundary
    height int
}

func largestRectangleArea(heights []int) int {
    stack := []Pair{}
    maxArea := 0

    for i, h := range heights {
        start := i
        for len(stack) > 0 && h < stack[len(stack)-1].height {
            // pop
            top := stack[len(stack)-1]
            stack = stack[:len(stack)-1]

            area := top.height * (i - top.index)
            maxArea = max(maxArea, area)
            start = top.index
        }
        stack = append(stack, Pair{index: start, height: h})
    }

    n := len(heights)
    for len(stack) > 0 {
        // pop
        top := stack[len(stack)-1]
        stack = stack[:len(stack)-1]

        maxArea = max(maxArea, top.height * (n-top.index))
    }
    return maxArea
}
```

要注意的部分是，因為 stack 最後還可能留有遞增的 histogram，需要清除，此時的 width 就是從底部 `len(heights)` 減去左邊界。

#### 853. Car Fleet

題目： 一條單行道上有多輛車，各有位置和速度。當後面的車追上前面的車時，不能超車，反而會以慢速一起以同一個車隊抵達終點。計算最終會有幾個車隊（Car Fleet）到達終點。

按位置由近終點到遠排序，計算每輛車抵達時間，這邊時間的算法，重溫國小數學：距離除以速度。接著使用 monotonic stack 找出「後車追不上前車」的組合：若後車的時間小於 stack.top （會比前車還要快到終點），則合併成一個車隊（不 push 進 stack）。反之，也就是說只有比前面車隊更晚抵達才會算數。

```go
func carFleet(target int, position []int, speed []int) int {
    n := len(position)

    type Car struct{ pos, spd int }
    cars := make([]Car, n)
    for i := range cars {
        cars[i] = Car{position[i], speed[i]}
    }
    sort.Slice(cars, func(i, j int) bool {
        return cars[i].pos > cars[j].pos
    })

    stack := []float64{} // arrive time
    for _, car := range cars {
        t := float64(target-car.pos) / float64(car.spd)

        if len(stack) == 0 || t > stack[len(stack)-1] {
            stack = append(stack, t)
        }
    }
    return len(stack)
}
```

## 上個禮拜去算命...
因緣際會下終於去朝見我婆婆那個很靈驗的算命奶奶！（還是要叫阿姨？我也都要 30 歲了）

結果奶奶居然說我現在也不要找工作...因為也無法保證找到的工作會比現在的好...

這倒是...我現在的工作 loading 輕、時間彈性、沒有開會、沒有加班、只要請假一定准，甚至 35 天的 congé + RTT 請完了想請無薪假也都准... 所有的精力跟時間都可以拿來進修跟培養興趣，我心心念念的 FIRE (financial indenpandent, retire early) 的生活可能也不過如此

### 不是，這樣還要刷題嗎
我接著問，那我這些準備還有意義嗎？還是都不要準備好啦（？）

奶奶說了可以，我帶著樂觀的心把她的回答當作是對我現在耕耘的肯定，過一陣子再回來找她吧。