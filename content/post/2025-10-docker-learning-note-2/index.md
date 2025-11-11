---
title: "Docker 映像檔"
description: 
date: 2025-10-11T18:20:36+02:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
tags:
    - Docker
    - Container
categories:
    - reading-note
    - dev
---

同樣的，今天的文章也是[《跟著 Docker 隊長，修練 22天就精通》](https://www.tenlong.com.tw/products/9789863126799)的讀書筆記，內容包含了第 3、4 章。

## 建立容器映像檔
Docker 根據 Dockerfile 的設定，將程式打包成一個映像檔。打包的時候，需要提供映像檔的名字，以及存在哪裡，`--tag` 用來設定映像檔的名稱。

```bash
docker image build --tag web-ping . # . 代表存在當前的目錄
```

接著就會一行一行的根據 dockerfile 的設定將目前的目錄 (context) 送到 Docker daemon 然後建置起來！ 

### 環境變數

我們可以在執行容器的時候透過指令 `--env` 或是 `-e` 改變環境變數。以書裡的例子，這邊 ping 的對象被設定為 `google.com`

```bash
docker container run --env TARGET=google.com diamol/ch03-web-ping
```

本機端會有一組預設環境變數，但這並不影響我們想要在容器內設定自定義的環境變數，換句話說，當同時執行兩個不同的容器，環境變數也可以分別設定。這是容器帶來的其中一個好處，將環境乾淨分割。

### Dockerfile
透過 dockerfile，應用程式具備可移植性。不管在哪裡的容器都可以執行該應用程式，build server 只需要安裝 docker 就可以同步不同硬體條件中的開發環境。

Dockerfile 沒有副檔名，裡面常用的命令包含 FROM, ENV, WORKDIR, COPY, CMD:
- `FROM` 載入映像檔
- `ENV` 以 key-value 的形式設置環境變數
- `WORKDIR` 在映像檔系統中根據作業系統建立一個目錄，並設定為工作目錄
- `COPY` 把檔案從本機複製到映像檔裡頭
- `CMD` 在容器啟動時自動執行
- `RUN` 在容器裡執行命令，命令的結果會留在映像層

書裡的範例如下
```dockerfile
# 以 diamol/node 作為基底映像檔，因為 diamol/node 安裝了執行該專案所需的 node.js 環境
FROM diamol/node
ENV TARGET="blog.sixeyed.com"
ENV METHOD="HEAD"
ENV INTERVAL=3000

WORKDIR /web-ping
COPY app.js .

CMD ["node", "/web-ping/app.js"]
```

### 多階段 Dockerfile
有的程式經歷多個階段才可以完成建置映像檔，每個階段都會由 `FROM` 開頭、並可以使用 `AS` 來命名。

書裡的範例包含了三個階段，雖然每個階段都是獨立執行，但可以把前一個階段的結果（檔案與目錄）複製起來。
```dockerfile
# build-stage
FROM diamol/base AS build-stage
RUN echo 'Building...' > /build.txt

# test-stage
FROM diamol/base AS test-stage
COPY --from=build-stage /build.txt /build.txt
RUN echo 'Testing...' > /build.txt

# final
FROM diamol/base
COPY --from=test-stage /build.txt /build.txt
CMD cat /build.txt
```

我們可以從 docker hub 裡找到自己專案所需的映像檔當作基礎檔案，像是帶有建置工具的、或是帶有 runtime 環境的映像檔。

> 多階段的建置也可以優化 CI/CD pipeline，如每個階段只需要複製該階段需要的文件、測試的內容與工具不會進入 production...等。透過這樣的操作，不但可以減少最後映像檔的大小、縮短啟動時間，映像檔中使用到的軟體更少也可以有效減少漏洞攻擊的風險。

如 java 的建置過程中，最終會把編譯好的 JAR 檔拿出來寫入映像檔，但使用直譯語言 JS 的 node.js 專案就會將執行環境與程式碼放入映像檔中。

## 映像層 Image Layers

當我們從 docker hub 下載 (pull) 一個映像檔時，會看到有很多檔案被下載下來，這些檔案被存在 Docker engine 快取裡。每個檔案分別是一個映像層。

一個映像檔會分成一層層的映像層，組裝在一起才是完整的容器檔案。這麼做的目的是當有很多個 share 相同環境的容器時，可以共用底層的映像層。像是前面指令的 `FROM diamol/node` 提供了一個 base OS 層與一個 node js 映像層。這樣的好處是當有很多檔案時，磁碟使用空間可以被節省起來。這些共用的映像層會被設為唯讀，以防修改後其他映像檔沒辦法使用。

使用 `history` 指令，可以看到映像檔的 metadata

```bash
docker image history web-ping
```

### 優化 Dockerfile

每一個 Dockerfile 的指令，會各自對應到一層映像層。在建置的時候，當修改了一個命令，這個命令以前的都需要重新執行。

因此可以使用這個映像層快取的特性來優化，以節省時間。不常改的放前面，常常改的放後面。

以剛剛的範例來說，可以優化成如下

```dockerfile
FROM diamol/node

CMD ["node", "/web-ping/app.js"]

ENV TARGET="blog.sixeyed.com" \
    METHOD="HEAD" \
    INTERVAL=3000

WORKDIR /web-ping
COPY app.js .
```

這樣一來，假設修改了 `app.js` 再重新 build，除了最後一層，其他都會使用快取的方式執行。

## 分散式應用程式

完整的應用程式通常會需要整合多個容器，如網頁服務，後端是一個應用程式、前端是一個、log 也可以是一個獨立的服務，那就會有好幾個容器彼此獨立運作並透過 docker 提供的虛擬網路互相溝通。

### Docker 虛擬網路

docker 在建立容器時會自動分配虛擬 IP 地址。我們可以使用 `docker network create nat` 來創造一個虛擬網路，或是在執行容器時使用參數 `--network` 將容器連接到虛擬網路，該網路上的任何容器就可以依據容器名稱互相溝通。

```bash
docker container run --name iotd -d -p 800:80 --network nat image-of-the-day
```
這個指令啟動一個容器並命名為 `iotd` 然後於後台持續 (`-d`) 執行 `image-of-the-day` 這個映像檔。接著把容器的連接埠 80 連接到本機 port 800，把服務連到 docker 提供的虛擬網路 `nat` 上，也就是說從本機打開 localhost:800 可以訪問容器的服務。


## 總複習
目前學到最基本最基本的指令包含跟容器相關的 `docker container`、跟映像檔相關的 `docker image`、建置映像檔需要的 `docker build`、以及最終在容器執行映像檔的 `docker run`。

### lab 4

這個階段要把以下 dockerfile 優化

```dockerfile
FROM diamol/golang:2e

WORKDIR /web
COPY index.html .
COPY go.mod main.go /web/

RUN go build -o /web/server
RUN chmod +x /web/server

CMD ["/web/server"]
ENV USER=sixeyed
EXPOSE 80
```

首先，我們可以先把單一階段分段成**多階段 build**：此處可以分成 build 跟 run。接著，為了最佳化最終的 image size，我們可以觀察有哪些東西是只在建構階段需要，就可以從 run 階段移除，只保留所需最小環境。這邊有一個小細節，`diamol/golang:2e` 就包含了完整 go 的工具庫，當在執行階段時可以改用 `diamol/base:2e` 以縮小映像檔尺寸。

```dockerfile
# build
FROM diamol/golang:2e AS builder

COPY go.mod main.go /go/
RUN go build -o /server
RUN chmod +x /server

# production
FROM diamol/base:2e

EXPOSE 80
CMD ["/web/server"]
ENV USER="sixeyed"

WORKDIR /web
COPY --from=builder /server .
COPY index.html .
```

另外， go image 的預設工作目錄是 `/go`，也就是說當執行 `RUN go build` 時，預設的 WORKDIR=/go/。優化後的文件結構是把需要編譯的檔案複製到 `/go/` 目錄下，Go 編譯器就可以找到 go.mod，不需要另外宣告 `WORKDIR`。

優化前的映像檔大小約 450 MB
```bash
miyalee@miyademac-mini lab % docker image ls -f reference=lab                             
REPOSITORY   TAG       IMAGE ID       CREATED         SIZE
lab          latest    a3ad6162d530   2 minutes ago   453MB
```

優化後縮減至僅剩 35MB !!!
```bash
miyalee@miyademac-mini lab % docker image ls -f reference=lab
REPOSITORY   TAG       IMAGE ID       CREATED         SIZE
lab          latest    f50f30c100ec   7 seconds ago   35.2MB
```

我們前面還有提到映像層會快取起來，當有文件修改時，對應到的那一層以及其之後的每一層會重新 build，因此我們會盡可能希望把常常改動的部分放到比較後面的階段。

假設對 index.html 有所修改，在初始版本的 dockerfile 中，從 `COPY index.html .`開始算共有 7 個步驟被影響，無法使用快取。而後面的優化版本，我們把 `COPY index.html .` 放到了最後，因此只有一個步驟需要重新執行。