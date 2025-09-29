---
title: "Docker 101"
description: 
date: 2025-09-29T15:36:15+02:00
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

之前的工作有碰過 Kubernetes (k8s)，不過對底層技術與實作並不熟悉，當時有主管帶著我飛因此天不怕地不怕。如今換了工作，目前的公司有超多 legacy code，很多軟硬體都偏老舊，算是我們還在數位轉型的階段，因此 dockerize 技術某種程度上變成了必要條件。在我前一份工作，我就對這個很感興趣，雖然目前不再是 devops，不知道目前的工作會不會有直接派上用場的機會，但總覺得學了準沒錯。

因此在最愛的天瓏書店購入了[《跟著 Docker 隊長，修練 22天就精通》](https://www.tenlong.com.tw/products/9789863126799)，結果第二章就覺得有點硬 XD 不做筆記的話會忘記。另外一提，作者是 Docker 技術的推廣者之一，還製作了與章節同步的[教學影片](https://www.youtube.com/playlist?list=PLXl_isu8qxvmDOAnUkG5x16LzBzGzY_Ww)，真的是佛心來著。（不過天知道我哪天為有閒有情致來看）

## 基本語法

### `docker run`

書裡的第一個語法是 `docker run diamol/ch02-hello-diamol`，但因為地端沒有這個映像檔，因此會去 **Docker hub** 下載。這是一個雲端倉庫服務，有點像 git clone 的感覺。

而一開始安裝的 Docker desktop 是用來提供本地 docker 環境、執行容器與管理地端映像檔的工具。當第一次執行 `docker run` 時，Docker desktop 會自動從 Docker hub 下載 image 檔案到本地，然後再執行容器。

---

如果想提供參數，`--tty` 或是簡寫 `-t`，用來分配一個偽終端機 (pseudo-TTY)，讓容器內的程式認爲他正在跟真實的終端機互動，提供像是彩色字體跟格式化的顯示，也支援互動性。通常會跟 `--interactive` 或 `-i` 共用，也可以再簡寫為 `-it`。想要跳出容器的終端機對話視窗時，只需要按 ctl+D 或是直接輸入指令 `exit` 即可。

> `docker container exec {CONTAINER ID} ...` 也可以進入容器 shell 執行指令

---

最後，可以透過更多參數在容器中執行一個網站

```
docker contaier run --detach --publish 8088:80 diamol/ch02-hello-diamol-web
```

- `--detach` 代表在背景啟動容器並顯示容器 ID，所以執行後會看到一串很長的 ID
- `--publish 8088:80` 配發值記得網路連接 port 給容器使用，也就是對外的公開 port 是 8088 但容器內部的 port 是 80

而容器跟實際電腦的 IP 也不一樣，每個容器都有自己專屬的 IP，也就是 docker 裡頭建立的虛擬 IP。電腦的 IP 是在連接 router 時分配到的，而容器的虛擬網路 IP 是 docker 分配的。所以電腦並不能直接連接到容器，而是要透過這個虛擬的 port 把流量轉到容器中。雖然瀏覽器連到的網址 `localhost:8088` 是地端電腦提供的，但網站內容是容器提供的，驗證了打包在容器裡的應用程式可以很容易被移植！｡:.ﾟヽ(*´∀`)ﾉﾟ.:｡

### `docker container ls`

使用這個指令可以看目前哪些容器正在執行，在 docker 指令操作時就是透過這邊的 `CONTAINER ID` 來進行辨識。

> 不過 `CONTAINER ID` 不需要全打，只要打部分就可以 (如前面兩個字)。

如果想要看完整的容器，包含已經 exited 的，可以加上參數 `--all`

---

### `docker container top {CONTAINER ID}`

印出某個指定容器的 process

---

### `docker container logs {CONTAINER ID}`

docker 會把應用程式的輸出存在 log 檔，這個指令可以把 log 叫出來

---

### `docker container inspect {CONTAINER ID}`

完整的輸出容器的設定，如系統路徑、執行命令、網路設定等等

---

### `docker container stats {CONTAINER ID}`

即時顯示容器正在使用多少資源，如 CPU、記憶體、網路流量等

我不確定怎麼退出，但狂按 ctl+C 就好像蠻有用的 (?)

---

### 刪除容器

```
docker container rm --force ${docker container ls --all --quiet}
```

這個指令會**強制刪除所有容器，包含正在運行與已經停止的**。刪除 container 可以透過指令，也可以直接把 docker desktop 退出，就自動會把在 run 的 container 刪掉，但不會刪除映像檔，所以下次回來還是可以再執行，不需要再次下載。

- `--all` 包含已經停止的
- `--quiet` 只顯示容器 ID
- `${}` 將內層指令的輸出作為外層指令的參數，像是算數裡小括號的概念
- `--force` 強制刪除，不管容器有沒有在執行

所以大括號裡會輸出一堆 ID，然後外部指令就會將這些容器一一刪除！！因為 rm 這個指令通常蠻危險的，因此有比較安全的版本 `docker container prune` 只刪除已經停止的容器。

## Docker Engine
**Docker engine** 是 Docker 的核心引擎 (daemon)，他負責實際的執行容器、管理映像檔、運行背景程式，並透過 API 進行互動。

最一開始下載的 **Docker Desktop** 是桌機的應用程式，包含一些圖形化介面，當然他內建了 docker engine 還有 **docker cli**。其他的 GUI 還有像是 **docker UCP** 以及 **Portainer**。

前面提到的 **Docker hub** 則跟本地 docker 沒有直接的關係，但是可以透過 CLI → API 在這邊分享或下載映像檔。

## Lab

蠻神奇的，docker 可以在 run 的時候在裡面換檔案，我猜應該還是有 build，但因為檔案很小，看不出來這個時間差是不是。

主要想考的是書上沒提到的指令，可以透過 `--help` 去看細節，可以得知從本機複製檔案到容器的語法是 `docker container cp <本機路徑> <容器ID>:<容器內路徑>`，反之亦然。舉例來說，把資料複製到地端目前位置

```
docker container cp fe8:/usr/local/apache2/htdocs/index.html .
```

也可以複製整個資料夾，如

```
docker container cp fe8:/usr/local/apache2/htdocs/ ./htdocs/
```
