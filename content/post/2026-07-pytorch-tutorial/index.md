---
title: "機器學習(0)——pyTorch"
description: NTUEE ML 2021 HW0
date: 2026-07-29T19:18:45+02:00
image: 
math: true
license: 
hidden: false
comments: true
draft: false
tags:
    - Machine Learning
    - PyTorch
categories:
    - machine-learning
---

經過約四個月的夏季 sprint，我結束了一個短暫但精彩 (?) 的創業之路，有空再來講創業的開發故事。

暑假期間決定好好來開展 Q3 的 12-week year，其中一個目標是把 NTU 李宏毅老師的機器學習作業完成。平常看課的影片有跟著讀書會，所以大致上的理解都還行，但因為課堂有很多數學、理論，感覺離實作好像很遙遠，但我一個純軟人！怎麼能放過練習開發的機會！

先從 PyTorch 的基本開始入門。

## 核心概念

使用 PyTorch 進行訓練時的核心邏輯很簡單，流程基本上就是：**把資料變成 tensor → 讓 tensor 流過模型 (forward) → 算出誤差 (loss) → 反向傳播算出梯度 (backward) → 更新參數 (optimizer.step())。**

以下工具都是圍繞著這個流程設計的。

### 建立 Tensor
神經網路的權重 (weight) 一開始必須是隨機的，而不是全部設成 0。

如果全部設成 0，每個神經元在第一次 forward 時輸出會一模一樣，backward 時梯度也一樣，導致所有神經元永遠學到同樣的東西（symmetry problem）。
用 `torch.randn`（常態分佈）而不是 `torch.rand`（均勻的隨機分佈）是因為常態分佈的初始化在數學上比較好控制權重的變異數，訓練起來比較穩定。

實務上 PyTorch 的 `nn.Linear` 等層其實已經用更講究的方法（如 Kaiming/Xavier init）初始化好了，通常不用自己手動 `randn`。

```python
pythontorch.randn(4, 5)   # 標準常態分佈 N(0,1): 平均值=0，標準差=1
torch.rand(4, 5)    # 均勻分佈 [0,1)，都是正數
torch.zeros(4, 5)   # 全為 0
torch.ones(4, 5)    # 全為 1
torch.tensor([1,2,3])  # 直接從 Python list/資料轉成 tensor
```

### 檢查與改變形狀
神經網路每一層對輸入的 shape 都有嚴格要求（例如 `nn.Linear(in_features, out_features)`），資料的維度只要對不上就會直接報錯。寫 model 時 90% 的 debug 時間都是在對 shape，所以養成習慣：每寫一行就 `print(x.shape)` 檢查。

```python
pythonx.shape    # 或 x.size()，看維度
x.view(2, 10)    # 改變形狀（需要記憶體連續）
x.reshape(2, 10) # 改變形狀（較安全，必要時會複製資料）
x.unsqueeze(0)   # 增加一個維度，例如 (5,) → (1,5)
x.squeeze()      # 移除大小為1的維度
x.permute(1,0)   # 交換維度順序（例如轉置的推廣版）
```

### Autograd（自動微分）
機器學習的本質是「調整參數讓 loss 變小」，而調整方向要靠梯度，也就是 loss 對每個參數的偏微分。

```python
pythonx = torch.tensor([2.0], requires_grad=True)
y = x ** 2
y.backward()
print(x.grad)  # dy/dx = 2x = 4.0
```

- `requires_grad=True`：告訴 PyTorch「這個 tensor 是需要被學習的參數，請幫我記錄它的計算過程」
- `loss.backward()`：沿著計算圖反向傳播，自動算出每個參數的梯度，存到 .grad 裡

### nn.Module

繼承 `nn.Module` 讓 PyTorch 自動幫你追蹤所有子層（self.layer1 等）裡的參數，這樣 `model.parameters()` 才能一次拿到所有要訓練的權重，交給 optimizer
`forward()` 定義資料怎麼流過模型

```python
import torch.nn as nn

class MyModel(nn.Module):
    def __init__(self):
        super().__init__()
        self.layer1 = nn.Linear(10, 5)
        self.relu = nn.ReLU()
        self.layer2 = nn.Linear(5, 1)

    def forward(self, x):
        x = self.layer1(x)
        x = self.relu(x)
        x = self.layer2(x)
        return x

model = MyModel()
output = model(x)  # 呼叫 model(x) 時會自動觸發 forward()
```

### Loss function
loss 定義了「模型現在有多差」，也是 backward 的起點。

```python
nn.MSELoss()          # 迴歸問題：預測值和真實值的均方誤差
nn.CrossEntropyLoss() # 分類問題：內部包含 softmax + negative log likelihood
nn.BCELoss()          # 二元分類：要自己先算 sigmoid
nn.BCEWithLogitsLoss()# 二元分類：內建 sigmoid，數值較穩定
```

選錯 loss 常見錯誤：
- 分類問題卻用 MSELoss：忽略了類別之間沒有「距離」的意義（類別3不會比類別1「大」）
- 用 CrossEntropyLoss 卻手動再加 softmax：會被算兩次 softmax，導致結果錯誤（CrossEntropyLoss 內部已經包含 softmax 了）

### Optimizer
optimizer 決定「怎麼用梯度更新參數」。

```python
optimizer = torch.optim.SGD(model.parameters(), lr=0.01)
optimizer = torch.optim.Adam(model.parameters(), lr=0.001)

optimizer.zero_grad()  # 清空上一輪殘留的梯度
loss.backward()        # 算梯度
optimizer.step()       # 依梯度更新參數
```

- SGD：最基本的梯度下降，通常需要手動調 learning rate 和 momentum，比較不穩但有時候收斂到更好的解
- Adam：自動幫每個參數調整學習率，收斂快、對超參數不敏感，是大多數作業的預設選擇

為什麼一定要 zero_grad()：PyTorch 預設會把每次 backward() 算出來的梯度累加而不是覆蓋，如果忘記清空，梯度會越滾越大

### Dataset + DataLoader
把資料切成一批一批(batch)餵給模型，`shuffle=True` 避免模型學到資料的排列順序、`batch_size` 平衡記憶體與訓練速度。

```python
dataloader = DataLoader(dataset, batch_size=32, shuffle=True)
```

### Train / Eval 模式切換
`nn.Dropout` 和 `nn.BatchNorm` 這類層在訓練和推論時行為不同。

```python
model.train()  # 訓練模式：啟用 dropout、batchnorm 用當前 batch 的統計量
model.eval()   # 驗證/測試模式：關閉 dropout、batchnorm 用訓練時累積的統計量

with torch.no_grad():  # 驗證/測試時不需要算梯度，省記憶體、加速
    output = model(x)
```

- 忘記 model.eval()：驗證時 dropout 還在隨機丟掉神經元，導致準確率忽高忽低、结果不穩定
- 忘記 torch.no_grad()：驗證時還在建立計算圖存梯度，浪費大量記憶體，容易 OOM

## 作業
接下來實際看看作業裡的程式長什麼樣子，連結我放在本文最後面

### `torch.max`
torch.max 在機器學習實作裡其實出現頻率蠻高的，但通常不是用「找最大值本身」，而是用它的第二種模式（回傳 value + index）。

```python
# 1. 整個 tensor 的最大值 (torch.max(input) → Tensor)
m = torch.max(x)

# 2. max along a dimension (torch.max(input, dim, keepdim=False, *, out=None) → (Tensor, LongTensor))
m, idx = torch.max(x,0) # 沿著 dim=0 找最大值
# or 
m, idx = torch.max(input=x,dim=0)
```
舉例來說，分類模型的輸出（logits）是每個類別的分數，不是「答案」本身。要知道模型「猜哪一類」，就要在「類別」維度上找分數最大的那個 index——這個 index 才是跟正確答案比對、算 accuracy 的東西。

### Common errors
#### different device error
```python
# wrong
model = torch.nn.Linear(5,1).to("cuda:0")
x = torch.Tensor([1,2,3,4,5]).to("cpu")
y = model(x)

# fixed
x = torch.Tensor([1,2,3,4,5]).to("cuda:0")
y = model(x)
print(y.shape)
```

在 PyTorch 中，device 指的是張量（tensor）或模型存放在哪個硬體上執行運算。CPU 和 GPU 是實體上不同的記憶體空間，就像兩台不同的電腦，資料不能直接相加。

常見的 device： "cpu": 一般 CPU，電腦預設 "cuda:0": 第一張 NVIDIA GPU "cuda:1": 第二張 NVIDIA GPU "mps": Apple Silicon GPU（M1/M2/M3）

#### mismatched dimensions error

```python
# wrong
x = torch.randn(4,5)
y= torch.randn(5,4)
z = x + y

# fixed
y = y.transpose(0,1)
z = x + y
print(z.shape)
```

#### cuda out of memory error
```python
# wrong
import torch
import torchvision.models as models
resnet18 = models.resnet18().to("cuda:0") # Neural Networks for Image Recognition
data = torch.randn(2048,3,244,244) # Create fake data (512 images)
out = resnet18(data.to("cuda:0")) # Use Data as Input and Feed to Model
print(out.shape)

# fixed
for d in data:
  out = resnet18(d.to("cuda:0").unsqueeze(0))
print(out.shape)
```
這樣做雖然省記憶體，但速度很慢，GPU 的優勢就是平行處理大量資料，一次只跑一張無法發揮 GPU 效能。實務上會用適當的 batch size（如 32）來平衡記憶體和速度。

假設有 2048 張圖片
```python
data = torch.randn(2048, 3, 224, 224)

batch_size = 32

for i in range(0, len(data), batch_size):
    batch = data[i : i + batch_size]          # 每次取 32 張
    out = resnet18(batch.to("cuda:0"))         # 一次處理 32 張
    print(f"batch {i//batch_size}: {out.shape}")
```

輸出：

```
batch 0:  torch.Size([32, 1000])
batch 1:  torch.Size([32, 1000])
...
batch 63: torch.Size([32, 1000])  ← 共 64 個 batch（2048/32）
```

#### mismatched tensor type

```python
# wrong
import torch.nn as nn
L = nn.CrossEntropyLoss()
outs = torch.randn(5,5)
labels = torch.Tensor([1,2,3,4,0])
lossval = L(outs,labels) # Calculate CrossEntropyLoss between outs and labels

# fixed
labels = labels.long()
lossval = L(outs,labels)
print(lossval)
```

`nn.CrossEntropyLoss` 內部會把 `labels` 當成類別索引（index），用來在 `outs` 的每一列中選出對應類別的分數去算 loss，因此要求 `labels` 是 `LongTensor`（整數型別）。但 `torch.Tensor([1,2,3,4,0])` 預設建立出來的是 `FloatTensor`，而不是 `LongTensor`，型別對不上就會噴 `RuntimeError: expected scalar type Long but found Float`。

用 `.long()` 把 `labels` 轉型成整數即可修正。

### More on dataset and dataloader
Dataset（資料集）可以想成是一堆資料用有組織的方式包起來。

具體來說 Dataset 是一個物件，可以用 dataset[i] 拿到第 i 筆資料（通常是 (x, y)，一組輸入和標籤）。它主要負責兩件事：
- 告訴你總共有幾筆資料（__len__）
- 給定一個 index，回傳那一筆資料（__getitem__），常常也包含讀檔、前處理（resize、normalize 等）

DataLoader（資料載入器）是一個可以走訪 dataset 的迭代器。

它包住 Dataset，負責：
- 把資料切成一批一批（batch_size）
- 每個 epoch 打亂順序（shuffle=True）
- 用多個 process 平行讀資料加速（num_workers）

關係大概是：Dataset 定義「有哪些資料、怎麼拿單一一筆」，DataLoader 定義「怎麼一批一批、有效率地把這些資料餵給模型」。對應到前面的段落：「把資料切成一批一批(batch)餵給模型」講的就是 DataLoader 的角色，Dataset 則是它背後包住的資料來源。

```python
import torch
import torch.utils.data 
class ExampleDataset(torch.utils.data.Dataset):
  def __init__(self):
    self.data = "abcdefghijklmnopqrstuvwxyz"
  
  def __getitem__(self,idx): # if the index is idx, what will be the data?
    return self.data[idx]
  
  def __len__(self): # What is the length of the dataset
    return len(self.data)

dataset1 = ExampleDataset() # create the dataset
dataloader = torch.utils.data.DataLoader(dataset = dataset1,shuffle = True,batch_size = 1)
for datapoint in dataloader:
  print(datapoint)
```

從 DataLoader 的視角，資料只是一串 index。DataLoader 並不真的知道「資料」長什麼樣子，它眼中的資料集其實就是 0, 1, 2, ..., N-1 這一串索引（index）。所以「打亂資料」這件事，本質上只是把這串 index 打亂順序，例如變成 3, 0, 4, 1, 2，而不是真的去搬動底層資料本身——這樣做效率更高，也不用真的複製一份資料。

此時，`torch.utils.data.DataLoader` 就是一個很好用的工具。DataLoader 需要 Dataset 提供兩個資訊

1. 資料總長度 (`__len__`)：DataLoader 要知道總共有幾筆資料，才知道 index 範圍是 0 到 N-1，也才知道切 batch 時要涵蓋多少筆
2. 依 index 拿資料 (`__getitem__`)：DataLoader 打亂完 index 順序後（例如這個 batch 要拿 [3, 0, 4]），它自己不知道怎麼把 index 換成真正的資料，必須請 Dataset 幫忙「翻譯」

## 參考
[作業連結](https://colab.research.google.com/github/ga642381/ML2021-Spring/blob/main/Pytorch/Pytorch_Tutorial.ipynb)
