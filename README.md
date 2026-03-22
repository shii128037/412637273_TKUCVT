# 412637273_TKUCVT
# W01｜虛擬化概論、環境建置與 Snapshot 機制

## 環境資訊
- Host OS：Windows 11
- VM 名稱：vct-w01-412637273
- Ubuntu 版本：
  No LSB modules are available.
  Distributor ID: Ubuntu
  Description:    Ubuntu 24.04.4 LTS
  Release:        24.04
  Codename:       noble
- Docker 版本：
  Docker version 29.3.0, build 5927d80
- Docker Compose 版本：
  Docker Compose version v5.1.1

## VM 資源配置驗證

| 項目 | VMware 設定值 | VM 內命令 | VM 內輸出 |
|---|---|---|---|
| CPU | 2 vCPU | `lscpu \| grep "^CPU(s)"` | CPU(s): 2 |
| 記憶體 | 4 GB | `free -h \| grep Mem` | Men: 3.8Gi  1.4Gi  525Mi  35Mi  2.2Gi  2.4Gi |
| 磁碟 | 40 GB | `df -h /` | /dev/sda2   40G  11G  27G 30%  / |
| Hypervisor | VMware | `lscpu \| grep Hypervisor` | Hypervisor vendor:   VMare |

## 四層驗收證據
- [ ] ① Repository：`cat /etc/apt/sources.list.d/docker.list` 輸出
      deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] http://download.docker.com/linux/ubuntu noble stable
- [ ] ② Engine：`dpkg -l | grep docker-ce` 輸出
      ii  docker-ce       5:29.3.0-1~ubuntu.24.04~noble
                amd64     Docker: the open-source application container engine
      ii  dockernse-cli   5:29.3.0-1~ubuntu.24.04~noble
                amd64     Docker CLI: the open-source application container engine
      ii  dockerice-rootless-extras  5:29.3.0-1~ubuntu.24.04~noble
                amd64     Rootless support for Docker.
- [ ] ③ Daemon：`sudo systemctl status docker` 顯示 active
      Active:active(running)since Sun 2026-03-22 21:15:42 CST; 3min 48s ago
- [ ] ④ 端到端：`sudo docker run hello-world` 成功輸出
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
17eec7bbc9d7: Pull complete
ea52d2000f90: Download complete
Digest: sha256:85404b3c53951c3ff5d40de0972b1bb21fafa2e8daa235355baf44f33db9dbdd
Status: Downloaded newer image for hello-world: latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
1. The Docker client contacted the Docker daemon.
2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading. 
4. The Docker daemon streamed that output to the Docker client, which sent it to your terminal.
   
To try something more ambitious, you can run an Ubuntu container with:  
$ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
https://hub.docker.com/

For more examples and ideas, visit:
https://docs.docker.com/get-started/
- [ ] Compose：`docker compose version` 可執行
      Docker Compose version v5.1.1

## 容器操作紀錄
- [ ] nginx：`sudo docker run -d -p 8080:80 nginx` + `curl localhost:8080` 輸出
      a9aa55fd6bb812b3ffaf2b94325442b297b478cf198f84d755174c5942f74d49
      curl: (35) OpenSSL/3.0.13: error: 0A00010B: SSL routines: :wrong version number
- [ ] alpine：`sudo docker run -it --rm alpine /bin/sh` 內部命令與輸出
/ # hostname
9f0e1cfa369b
/ # cat /etc/os-release
NAME= "Alpine Linux"
ID=alpine
VERSION_ID=3.23.3
PRETTY_NAME="Alpine Linux v3.23"
HOME_URL="https://alpinelinux.org/"
BUG_REPORT_URL=https://gitlab.alpinelinux.org/alpine/aports/-/issues"
/ #Ls /
bin  etc  lib  mnt  proc  run  srv  tmp  var
dev  home  media  opt  root  sbin  sys  usr
/ # whoami
root
/ # exit
- [ ] 映像列表：`sudo docker images` 輸出
IMAGE                 ID             DISK USAGE    CONTENT SIZE    EXTRA
alpine: latest        25109184c71b       13.1MB          3.95MB
hello-world: latest   85404b3c5395       25.9kB          9.52kB
nginx: latest         dec7a90bd097        240MB         65. 8MB

## Snapshot 清單

| 名稱 | 建立時機 | 用途說明 | 建立前驗證 |
|---|---|---|---|
| clean-baseline | （時間點）2026/3/22 21:57:10 | （此節點代表的狀態）建立第一個可回復基線。建 snapshot 之前必須先確認環境健康。 
| （列出建點前做了哪些驗證）hostnamectl
ip route
sudo docker --version
docker compose version
sudo systemctl status docker --no-pager
sudo docker run --rm hello-world |
| docker-ready | （時間點）2026/3/22 22:05:28 | （此節點代表的狀態）節省重複部署的時間 
| （列出建點前做了哪些驗證）sudo systemctl status docker --no-pager
sudo docker run --rm hello-world
sudo docker images    # 確認 nginx、alpine 映像都在 |

## 故障演練三階段對照

| 項目 | 故障前（基線） | 故障中（注入後） | 回復後 |
|---|---|---|---|
| docker.list 存在 | 是 | 否 | （填入）是 |
| apt-cache policy 有候選版本 | 是 | 否 | （填入）是 |
| docker 重裝可行 | 是 | 否 | （填入）是 |
| hello-world 成功 | 是 | N/A | （填入）是 |
| nginx curl 成功 | 是 | N/A | （填入）是 |

## 手動修復 vs Snapshot 回復

| 面向 | 手動修復 | Snapshot 回復 |
|---|---|---|
| 所需時間 | （你的實測）約 30 秒 | （你的實測）約 1～2 分鐘（含開關機） |
| 適用情境 | （你的判斷）小錯誤（單一檔案、設定明確） | （你的判斷）大規模錯誤、不確定改動內容 |
| 風險 | （你的判斷）修錯或漏修設定，可能造成更大問題 | （你的判斷）幾乎無風險，但會遺失未保存變更 |

## Snapshot 保留策略
- 新增條件：每次重大變更（安裝 Docker、修改系統設定）且功能驗證成功後建立
- 保留上限：最多保留 3 個 Snapshot
- 刪除條件：新 Snapshot 建立後，且舊 Snapshot 已無回復需求時刪除最舊版本

## 最小可重現命令鏈
（列出讓他人能重現故障注入與回復驗證的命令序列）
/# 基線確認
ls /etc/apt/sources.list.d/
apt-cache policy docker-ce | head -10

/# 注入故障（移除 docker repo）
sudo mv /etc/apt/sources.list.d/docker.list /etc/apt/sources.list.d/docker.list.broken
sudo apt update

/# 驗證故障
apt-cache policy docker-ce | head -10
sudo apt -y install docker-ce

/# 手動修復
sudo mv /etc/apt/sources.list.d/docker.list.broken /etc/apt/sources.list.d/docker.list
sudo apt update

/# Snapshot 回復（操作：VM → Revert）

/# 回復後驗證
sudo docker run --rm hello-world
sudo docker run -d -p 8080:80 nginx
curl http://localhost:8080

## 排錯紀錄
- 症狀：無法安裝 docker-ce，apt 顯示找不到候選版本
- 診斷：（你首先查了什麼？）
  使用 ls /etc/apt/sources.list.d/ 發現 docker.list 消失
  使用 apt-cache policy docker-ce 確認沒有 repository
- 修正：（做了什麼改動？）
  將 docker.list.broken 改回 docker.list 並重新 apt update
- 驗證：（如何確認修正有效？）
  apt-cache policy docker-ce 出現 Candidate
  docker run hello-world 成功
  curl localhost:8080 正常回應 nginx 頁面

## 設計決策
（說明本週至少 1 個技術選擇與取捨）
本次選擇使用 Snapshot 作為主要回復機制，而非完全依賴手動修復。

取捨說明：
優點：
可快速回復到「已驗證正確」的狀態
避免人為操作錯誤累積
適合教學與實驗環境反覆測試
缺點：
需要額外儲存空間
回復會遺失回復點之後的變更
